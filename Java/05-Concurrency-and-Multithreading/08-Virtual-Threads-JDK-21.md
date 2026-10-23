# Virtual Threads (JDK 21)

**Target level:** Advanced → Senior  
**Version status:** Virtual threads became a permanent feature in Java 21 (JEP 444).

Virtual threads make thread-per-task programming practical for many high-concurrency, blocking workloads. They improve scalability by reducing the cost of representing waiting tasks, but they do not make CPU work faster, remove downstream limits, or eliminate the need for cancellation, timeouts, and backpressure.

## 1. Virtual thread fundamentals

### Q1. What is a virtual thread? *(Intermediate)*

A virtual thread is a Java thread scheduled by the Java runtime rather than being permanently tied one-to-one to an operating-system thread. The runtime can suspend a virtual thread during supported blocking operations and use its carrier platform thread for other work.

Virtual threads implement Thread's programming model, so many existing thread-per-request designs can be adapted. Their low cost does not mean an unlimited number of concurrent database queries or remote requests is safe.

### Q2. When did virtual threads become permanent? *(Intermediate)*

Virtual threads became permanent in Java 21 through JEP 444 after preview releases in Java 19 and 20. In Java 21 they can be used without preview flags.

State the JDK version in interview answers. APIs and diagnostics may evolve in later releases, so do not mix Java 21 guarantees with behavior documented only for a newer runtime.

### Q3. How do you create virtual threads? *(Intermediate)*

Java 21 provides Thread.startVirtualThread and Thread.ofVirtual builder APIs, and Executors.newVirtualThreadPerTaskExecutor creates an executor that starts a new virtual thread per task.

    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        Future<String> result = executor.submit(() -> fetchData());
        System.out.println(result.get());
    }

The executor's close behavior and task lifetime should be understood. The example waits for the result; in a real service, use bounded deadlines and appropriate error handling.

### Q4. What workloads benefit most from virtual threads? *(Advanced)*

Virtual threads are especially useful for high-concurrency workloads where tasks spend much time waiting on blocking I/O, such as request handlers making remote calls. They allow a straightforward thread-per-task model without requiring one platform thread for every waiting task.

CPU-bound workloads are still limited by available CPU. Virtual threads do not speed up computation, and high concurrency can overwhelm databases, connection pools, rate limits, or remote services unless those resources are bounded.

### Q5. Should virtual threads be pooled like platform threads? *(Advanced)*

Generally, virtual threads are intended to be cheap and created per task rather than pooled to amortize thread creation. Pooling virtual threads can complicate task ownership and may not provide the resource limit people expect.

Pool or limit the scarce resource, such as database connections or concurrent calls to a downstream API, rather than pooling virtual threads as if they were expensive platform threads.

### Q6. What is a carrier thread? *(Advanced)*

A carrier is a platform thread that executes a virtual thread when it is scheduled. A virtual thread may run on different carrier threads over its lifetime. The runtime manages scheduling and continuation behavior.

Do not rely on thread-local state attached to a carrier to represent a virtual thread's logical request. Use the Java thread APIs and context propagation mechanisms appropriate to the application.

## 2. Blocking and pinning

### Q7. What does pinning mean for virtual threads? *(Advanced)*

Pinning occurs when a virtual thread cannot unmount from its carrier during a blocking operation in certain runtime circumstances, such as while executing a synchronized region or native/foreign call in the relevant Java 21 implementation. This can reduce scalability if many tasks block while pinned.

JDK behavior evolves, so verify the exact release's documentation and JEPs. Keep synchronized regions short and avoid blocking I/O inside them when it can cause many virtual threads to pin carriers.

### Q8. Should all synchronized blocks be replaced for virtual threads? *(Advanced)*

No. Synchronized remains a correct and useful mutual-exclusion mechanism. The concern is blocking while pinned in scenarios that affect carrier availability, not synchronization itself. Short critical sections protecting in-memory state are usually appropriate.

Do not mechanically replace synchronized with ReentrantLock. Measure pinning and throughput, and focus on long/blocking operations within synchronized regions where the target JDK's behavior makes it relevant.

### Q9. Do virtual threads make blocking I/O non-blocking? *(Intermediate)*

The programming model remains blocking, but the runtime can often unmount a virtual thread while supported operations wait, allowing the carrier to execute other virtual threads. The API is still synchronous from the task's perspective.

Not every operation necessarily unmounts, and native libraries or unsupported blocking behavior may pin or occupy carriers. Test the real driver and library stack.

### Q10. Are thread locals appropriate with virtual threads? *(Advanced)*

ThreadLocal works with virtual threads, but creating enormous numbers of virtual threads each with large thread-local values can increase memory usage. Thread-local state may also be unsuitable for request context if the architecture expects explicit propagation across asynchronous boundaries.

Use thread locals deliberately and keep values small. Scoped Values were a preview feature in Java 21, not a finalized Java 21 feature; do not describe them as permanent in that release.

## 3. Limits and production design

### Q11. Do virtual threads remove the need for backpressure? *(Senior)*

No. They reduce the cost of waiting tasks but do not increase the capacity of databases, downstream services, file descriptors, or external rate limits. Unbounded task submission can still create memory pressure and overload dependencies.

Use semaphores, connection pools, bounded admission, rate limiting, deadlines, and rejection policies to protect scarce resources. Measure concurrency at each dependency boundary.

### Q12. How should virtual threads interact with JDBC connection pools? *(Senior)*

A virtual thread can block while waiting for a JDBC connection, but the connection pool still has a finite number of connections. If too many tasks wait for a small pool, latency and memory usage can grow even if thread overhead is low.

Set pool size based on database capacity, use acquisition timeouts, limit concurrent database work when necessary, and monitor wait time. Do not increase the connection pool just because virtual threads make more requests possible.

### Q13. How do you cancel a virtual-thread task? *(Advanced)*

Use interruption and structured task ownership or Future cancellation as appropriate. Virtual threads follow cooperative interruption semantics; code that ignores interruption or waits in a non-interruptible native operation may not stop promptly.

Design blocking operations with timeouts and cleanup. Cancellation does not roll back an external side effect that already occurred.

### Q14. What should be measured before adopting virtual threads? *(Senior)*

Measure throughput, tail latency, heap usage, number of live virtual threads, downstream concurrency, connection-pool wait time, pinning diagnostics, and CPU utilization. Compare with the existing platform-thread or reactive implementation under realistic load.

A high virtual-thread count alone is not a problem, but a high count of tasks waiting on a saturated dependency can signal overload. Optimize the resource bottleneck, not just thread count.

### Q15. When might a reactive stack still be appropriate? *(Senior)*

Reactive APIs can remain useful when non-blocking end-to-end processing, backpressure propagation, streaming, or integration with an established reactive ecosystem is important. Virtual threads simplify blocking code but do not automatically provide reactive-stream demand semantics.

Choose based on end-to-end library support, operational expertise, throughput/latency requirements, and complexity. Avoid mixing blocking calls into event-loop threads; virtual threads are a separate execution model.

## Official references

1. [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
2. [Java SE 21 API — Thread](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html)
3. [Java SE 21 API — Executors](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Executors.html)
4. [JEP 446 — Scoped Values (Preview in Java 21)](https://openjdk.org/jeps/446)
5. [Java SE 21 Guide — Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

## Chapter checklist

- [ ] Explain virtual threads versus platform threads and carrier threads.
- [ ] Choose thread-per-task for suitable blocking workloads.
- [ ] Understand pinning concerns in the Java 21 release context.
- [ ] Preserve resource limits, timeouts, cancellation, and backpressure.
- [ ] Distinguish finalized virtual threads from preview scoped values.

## Worked examples: virtual threads and resource limits

### Example 1. Use one virtual thread per blocking task

~~~java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> response = executor.submit(() -> callBlockingService());
    System.out.println(response.get());
}
~~~

The virtual-thread-per-task executor is available in Java 21. The executor is AutoCloseable; closing it waits for submitted tasks to finish. Production code still needs explicit timeouts, error handling, cancellation, and downstream concurrency limits.

### Example 2. Virtual threads do not increase database capacity

A service may accept many more waiting tasks with virtual threads, but a database pool still has a finite number of connections. Bound concurrent database work with the connection pool or a semaphore, apply deadlines, and measure queue wait. Otherwise the application may move overload from the thread pool to the database queue.

### Example 3. Avoid thread-per-task as a substitute for admission control

Virtual threads make blocking tasks cheaper than one platform thread per task, but they do not make CPU-bound work free or remove memory costs for task state. Limit incoming work, bound retained queues and payloads, and protect downstream dependencies. Diagnose pinning and runtime behavior using documentation for the exact JDK release.
