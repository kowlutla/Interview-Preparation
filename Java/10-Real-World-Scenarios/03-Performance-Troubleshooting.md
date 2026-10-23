# Java Performance Troubleshooting Scenarios

### Scenario 1. p99 latency increases while average latency is stable

Check tail behavior, queueing, dependency outliers, GC pauses, and lock contention. Segment by endpoint and trace the slow requests.

### Scenario 2. Throughput falls after adding more workers

More workers may increase context switching, contention, or downstream overload. Compare active threads, queue wait, CPU, connection pools, and service-time distribution.

### Scenario 3. CPU is low while request latency is high

Investigate I/O waits, locks, connection acquisition, queueing, and downstream calls. More threads may worsen the bottleneck.

### Scenario 4. CPU is high but throughput does not rise

Look for hot algorithms, spinning, contention, retries, GC/JIT work, or saturated CPU capacity. Profile hot threads before increasing pool sizes.

### Scenario 5. A new stream pipeline allocates more than a loop

Check boxing, intermediate collections, collector choices, and pipeline shape. Benchmark with JMH and compare equivalent semantics.

### Scenario 6. A regular expression causes latency spikes

Check input length, pathological backtracking, repeated Pattern compilation, and untrusted input. Bound input and use a safer pattern or parser.

### Scenario 7. Logging causes a throughput regression

Inspect synchronous appenders, excessive string construction, high-volume logs, and serialization of large objects. Use structured logging, appropriate levels, and asynchronous logging only with a defined loss/shutdown policy.

### Scenario 8. Database CPU is low but application connection wait is high

Connections may be held too long, leaked, or used during remote calls. Inspect pool metrics and transaction duration before increasing the pool.

### Scenario 9. Latency increases as queue depth rises

The queue is accumulating work faster than consumers can process it. Bound queue capacity, add backpressure, and fix the constrained stage.

### Scenario 10. A microbenchmark says one version is faster but production is slower

The benchmark may not represent workload, warmup, data shape, concurrency, or dependencies. Compare profiles and load-test the actual service path.

### Scenario 11. Parallel streams reduce performance

Work may be too small, blocking, order-sensitive, or contended; splitting/combining overhead can dominate. Benchmark and compare sequential execution.

### Scenario 12. A cache improves hit latency but worsens p99

Look for synchronized hot keys, expensive loaders, eviction contention, stale refreshes, and unbounded cache maintenance. Measure hit/miss, load duration, and lock wait.

### Scenario 13. JSON serialization dominates CPU

Inspect payload size, reflection/metadata lookup, repeated object conversion, and unnecessary fields. Cache safe metadata, reduce payloads, and benchmark the serializer.

### Scenario 14. Startup is slow but requests are fast after warmup

Inspect class loading, framework scanning, static initializers, JIT warmup, and dependency initialization. Optimize startup separately if cold start matters.

### Scenario 15. A minor code change causes allocation spikes

Compare allocation profiles and object lifetimes, then inspect new collections, boxing, lambdas/captures, buffers, and serialization. Verify whether the allocation is harmful before rewriting code.

### Scenario 16. Throughput is high but error rate rises under load

The system may be beyond a dependency's safe capacity or retries may amplify failures. Inspect saturation, timeouts, retries, and rejection metrics; establish load shedding and backpressure.

### Scenario 17. Latency worsens after increasing heap

A larger heap may change GC behavior, container headroom, or collection timing; it will not fix a leak or CPU bottleneck. Compare live set, pauses, RSS, and allocation under the same workload.

### Scenario 18. Performance changes between JDK builds

Compare vendor/build, flags, default collector, dependencies, container detection, JIT, and runtime behavior. Run reproducible benchmarks on both versions.

### Scenario 19. Response body processing creates large temporary arrays

Avoid reading unbounded content fully into memory. Stream in bounded chunks, enforce size limits, and apply backpressure.

### Scenario 20. Service saturates only during downstream slowness

Threads and queues accumulate while waiting on the dependency. Add deadlines, bulkheads, bounded queues, and a clear fallback/rejection policy.

## Measurement checklist

Record workload, data distribution, concurrency, CPU/memory limits, JDK build, flags, throughput, p50/p95/p99, errors, allocation, GC, queueing, and downstream latency before and after every change.
