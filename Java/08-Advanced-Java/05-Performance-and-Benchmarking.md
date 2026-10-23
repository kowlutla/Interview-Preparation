# Java Performance and Benchmarking

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Performance work starts with defining the symptom and workload. Java performance can be limited by CPU, allocation/GC, lock contention, I/O, downstream capacity, serialization, or queueing. Good engineers form a hypothesis, measure it, and compare a reproducible baseline instead of tuning blindly.

## 1. Measurement fundamentals

### Q1. What is the difference between latency and throughput? *(Beginner)*

Latency is the time taken for an operation or request; throughput is the amount of work completed per unit time. A system can have high throughput and poor tail latency, or low latency at low load but collapse under concurrency.

Define the target SLO and measure percentiles, not only averages. Always state workload, concurrency, payload sizes, and resource limits when comparing results.

### Q2. Why are p95 and p99 important? *(Intermediate)*

Percentiles describe tail behavior: p99 is the latency at or below which 99% of observations fall. A small fraction of slow requests can dominate user experience and can be hidden by the mean.

Use enough observations and segment by endpoint/dependency. Correlate tail latency with queues, GC, lock contention, and downstream latency.

### Q3. What is JMH? *(Intermediate)*

JMH is the OpenJDK Java Microbenchmark Harness designed for JVM microbenchmarks. It handles many issues such as warmup, forks, result consumption, and benchmark execution structure.

JMH cannot make a poorly designed benchmark representative automatically. Choose realistic data and operations, avoid benchmarking only setup, and understand whether the benchmark is CPU-bound, allocation-bound, or dominated by external work.

### Q4. Why can naive System.nanoTime benchmarks mislead? *(Advanced)*

A simple timing loop may be affected by JIT warmup, dead-code elimination, constant folding, garbage collection, CPU frequency changes, background load, and insufficient repetitions. nanoTime measures elapsed time but does not control the experiment.

Use JMH for microbenchmarks and a load-testing tool for service-level behavior. Measure allocation and tail latency where relevant, not just average operation time.

### Q5. What is warmup? *(Intermediate)*

Warmup lets class loading, profiling, JIT compilation, caches, and runtime state stabilize before measurement. It matters for steady-state workloads, but startup latency must be measured separately when cold starts are part of the product.

Do not discard startup cost if the real deployment is short-lived or serverless. Benchmark the lifecycle that matches the product requirement.

## 2. CPU and allocation

### Q6. How do you find a CPU hotspot? *(Advanced)*

Use CPU profiling or JFR to identify hot methods and threads, then correlate them with workload and code paths. A thread dump alone gives stack snapshots but does not accurately attribute CPU time.

Look for algorithms with poor complexity, repeated parsing, regex work, excessive retries, serialization, lock contention, and unnecessary allocations. Optimize the dominant measured path.

### Q7. What is allocation rate, and why does it matter? *(Advanced)*

Allocation rate measures how quickly objects are created. High allocation can increase GC frequency and memory bandwidth usage even if most objects die quickly and there is no leak.

Use allocation profiling to identify hot allocation sites. Avoid eliminating allocations indiscriminately; simpler code can be faster if the JIT optimizes it well.

### Q8. What is the difference between allocation churn and a memory leak? *(Intermediate)*

Allocation churn creates many short-lived objects that are reclaimed, while a leak retains objects that are no longer needed. Churn often appears as high allocation/GC activity; a leak appears as growing retained live state.

Use allocation profiles for churn and heap dumps/reference paths for leaks. They can coexist and should be investigated separately.

### Q9. How can boxing affect performance? *(Intermediate)*

Boxing converts primitives to wrapper objects, which can add allocation, indirection, and GC pressure when not optimized away. Stream<Integer> can incur boxing where IntStream avoids it.

Use primitive-specialized APIs when they fit a measured hot path. Do not assume every wrapper operation allocates; the JIT may optimize some cases.

### Q10. How can algorithmic complexity dominate micro-optimization? *(Intermediate)*

An O(n²) algorithm can become unusable as input grows even if each operation is fast. Improving an algorithm to O(n log n) or O(n) often produces a much larger gain than changing syntax or a JVM flag.

Measure input sizes and asymptotic growth. Optimize the algorithm and data structure before low-level tuning unless evidence shows a constant-factor bottleneck dominates.

## 3. Concurrency and memory performance

### Q11. How does lock contention affect throughput? *(Advanced)*

Contended locks serialize otherwise parallel work and add waiting, context switching, and cache-coherence costs. A single hot lock can cap throughput regardless of CPU count.

Profile blocked time and critical-section duration. Reduce shared mutable state, partition state, shorten critical sections, or use suitable concurrent structures when the invariant permits it.

### Q12. Why can increasing thread count reduce throughput? *(Advanced)*

More threads can increase context switching, memory pressure, lock contention, queueing, and downstream overload. For CPU-bound work, runnable threads beyond available processing capacity often add overhead without adding useful parallelism.

Size pools around the constrained resource and measure. Blocking workloads may need more concurrency, but only if downstream capacity and queue limits support it.

### Q13. How do virtual threads affect performance analysis? *(Advanced)*

Virtual threads reduce the cost of representing many waiting tasks, but they do not increase CPU or downstream service capacity. Metrics should include active task count, dependency wait time, connection pool saturation, and pinning diagnostics where relevant.

Do not interpret a large virtual-thread count alone as a failure. Determine whether tasks are making progress or waiting behind a scarce resource.

## 4. Tuning workflow

### Q14. How do you decide whether to tune the GC? *(Senior)*

First verify that GC contributes materially to the SLO violation using logs, JFR, pause percentiles, allocation rate, and live-set data. If latency is dominated by database waits or lock contention, changing the collector is unlikely to fix it.

Benchmark supported collectors and heap configurations against the target JDK and workload. Track throughput, tail latency, CPU, and total memory.

### Q15. What is a good performance optimization workflow? *(Senior)*

1. Define the metric and workload.
2. Capture a reproducible baseline.
3. Profile to locate the dominant cost.
4. Form one hypothesis and change one variable.
5. Re-run the same workload and compare distributions.
6. Add a regression test/benchmark or metric.
7. Document trade-offs and rollback criteria.

This prevents “optimization” that improves a microbenchmark but harms production latency or maintainability.

### Q16. How should a backend service be load-tested? *(Senior)*

Use realistic traffic mix, payload sizes, concurrency, warmup, data cardinality, downstream dependencies, and resource limits. Measure throughput, p50/p95/p99 latency, errors, saturation, queueing, GC, CPU, and memory.

A closed-loop test can hide overload because clients slow down when responses slow; consider whether an open-loop arrival model better matches production. Include failure and recovery behavior.

### Q17. What are common performance anti-patterns? *(Senior)*

Common issues include optimizing without profiling, using ad hoc microbenchmarks, increasing pools without measuring downstream capacity, using parallel streams for blocking work, unbounded queues, excessive logging/serialization, and retaining unnecessary object graphs.

Fix the measured bottleneck and verify system-level impact. Performance is a whole-system property, not just a method's execution time.

## Official references

1. [JMH — Java Microbenchmark Harness](https://openjdk.org/projects/code-tools/jmh/)
2. [Java SE 21 Flight Recorder documentation](https://docs.oracle.com/en/java/javase/21/jfapi/index.html)
3. [Java SE 21 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
4. [Java SE 21 HotSpot performance enhancements](https://docs.oracle.com/en/java/javase/21/vm/java-hotspot-virtual-machine-performance-enhancements.html)

## Chapter checklist

- [ ] Separate latency, throughput, and tail behavior.
- [ ] Use JMH for microbenchmarks and realistic load tests for services.
- [ ] Profile CPU, allocation, lock contention, and I/O before tuning.
- [ ] Distinguish allocation churn from retained leaks.
- [ ] Verify improvements against system-level SLOs.
