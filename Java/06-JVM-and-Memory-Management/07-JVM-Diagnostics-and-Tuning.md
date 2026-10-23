# JVM Diagnostics and Tuning

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Effective JVM troubleshooting begins with a symptom, a reproducible baseline, and evidence. The goal is to identify the constrained resource—CPU, allocation/GC, locks, threads, native memory, I/O, or a downstream dependency—before changing runtime flags.

## 1. Diagnostic tools

### Q1. What is jcmd? *(Intermediate)*

jcmd sends diagnostic commands to a running JVM, enabling operations such as thread dumps, JFR control, VM flags, and certain heap/GC diagnostics. Available commands depend on the target JDK and JVM.

Prefer documented jcmd commands over unsupported internal mechanisms. Capture the exact JDK version and command output because diagnostic capabilities can change between releases.

### Q2. What is jstack? *(Intermediate)*

jstack prints Java thread stack traces and lock-related information for a process, depending on platform and JDK version. jcmd Thread.print is another common approach for thread dumps.

Thread dumps are snapshots. Take multiple dumps over time to identify stable blocking, deadlocks, repeated stack patterns, and progress.

### Q3. What is Java Flight Recorder? *(Intermediate)*

JFR is a low-overhead event recording facility for JVM and application events, including allocation, GC, thread activity, locks, exceptions, and other configured events. Recordings can be analyzed with Java Mission Control or other supported tooling.

JFR is useful for time-correlated evidence, not just one-time snapshots. Event settings and overhead depend on configuration and workload.

### Q4. What is Java Mission Control? *(Intermediate)*

Java Mission Control is a tool for analyzing JFR recordings and JVM performance data. It can help correlate CPU, allocation, GC, locks, and thread behavior over time.

Use the tool version and JDK support appropriate to the recording. Do not assume every event or UI feature exists in every JDK distribution.

### Q5. What is jmap? *(Intermediate)*

jmap is a diagnostic utility associated with heap and memory inspection tasks, though recommended commands and capabilities vary by JDK. Heap dumps can also be triggered through documented JVM options or diagnostic commands.

Heap dumps may pause the application and contain sensitive data. Plan capture, storage, access control, and disk space before using them in production.

### Q6. What is jstat? *(Intermediate)*

jstat reports selected JVM statistics such as garbage collection and class-loading counters. It can provide a quick view of trends but is not a substitute for detailed logs, JFR, or heap analysis.

Use it to establish a symptom and then collect deeper evidence. Avoid inferring a complete diagnosis from one counter.

## 2. GC and memory tuning

### Q7. What is unified logging? *(Advanced)*

Modern JDKs use unified JVM logging configured with -Xlog selectors. It can emit GC, class-loading, safepoint, and other diagnostic categories. Syntax and available tags should be checked against the target JDK.

For a production service, enable the minimum useful diagnostics with rotation and retention limits. Logs that fill the disk or overwhelm operators can create a new incident.

### Q8. How do you select a garbage collector? *(Senior)*

Define the target: throughput, tail latency, heap size, CPU budget, and platform support. Test supported collectors under realistic workload and load. Consider concurrent GC CPU and memory overhead, not just pause time.

Use the default collector as a baseline unless evidence suggests otherwise. Record JDK version and flags so performance comparisons remain reproducible.

### Q9. How do you tune heap size? *(Senior)*

Set heap limits from live-set size, allocation rate, GC behavior, container limits, and native-memory headroom. A larger heap can reduce collection frequency but may increase memory footprint and does not fix leaks or unbounded queues.

Measure after changing heap parameters. Ensure the process's total resident memory stays within the deployment limit.

### Q10. What is the difference between -Xms and -Xmx? *(Intermediate)*

-Xms sets the initial/minimum heap sizing target as defined by the JVM option, and -Xmx sets the maximum Java heap size. Exact ergonomics and how the heap grows depend on the runtime and collector.

Setting them equal can reduce resizing variability but commits memory differently and may be unsuitable in constrained containers. Choose based on startup and memory-budget requirements.

## 3. CPU and latency

### Q11. How do you investigate high CPU usage? *(Senior)*

Identify hot threads and CPU profiles, then correlate stacks with request throughput, JIT behavior, allocation, and dependency activity. A thread dump shows stacks but does not by itself provide accurate CPU attribution; use a profiler or JFR CPU events.

Separate application CPU from GC/JIT CPU. Avoid increasing thread counts for CPU-bound workloads without evidence, because additional runnable threads can worsen contention.

### Q12. How do you investigate high latency with low CPU? *(Senior)*

Look for blocked threads, lock contention, connection-pool waits, network latency, disk I/O, queue wait time, and external dependencies. Low CPU can indicate the service is waiting rather than computing.

Use distributed traces, thread dumps, queue metrics, and pool metrics together. More threads may only increase pressure on the constrained dependency.

### Q13. How do you interpret p95 and p99 latency? *(Intermediate)*

p95 is the latency below which 95% of measured requests fall; p99 is the corresponding 99th percentile. Tail percentiles reveal outliers that averages can hide.

Measure enough requests over a relevant time window and segment by endpoint or dependency. Correlate percentiles with GC pauses, queueing, and load to identify the source of tail latency.

### Q14. What is warmup in JVM benchmarking? *(Intermediate)*

Warmup allows class loading, profiling, JIT compilation, and runtime state to stabilize before measurement. Cold-start workloads may never reach steady state, so warmup should match the actual product goal rather than be blindly discarded.

Use separate cold-start and steady-state measurements when both matter. JMH handles many benchmark mechanics but still requires representative benchmark design.

## 4. Production incident playbooks

### Q15. What should be collected for a JVM performance incident? *(Senior)*

Collect the exact JDK/build, JVM flags, container limits, deployment change history, latency/throughput metrics, CPU and RSS, GC logs, JFR, thread dumps, pool/queue metrics, and dependency latency. Preserve the time range around the incident.

Capture evidence before restart when safe. Avoid changing multiple flags simultaneously because it makes cause and effect hard to establish.

### Q16. How do you distinguish a memory leak from a temporary spike? *(Senior)*

Compare live-set size after full/major collection where meaningful, heap dumps over time, retained object paths, and allocation profiles. A temporary burst may return to a stable baseline; a leak tends to show persistent retained growth, though the pattern can be noisy.

Use repeated observations under similar workload and avoid relying on a single heap graph. Determine whether growth is caused by intended cache warmup, traffic changes, or unintended retention.

### Q17. How should JVM tuning changes be rolled out? *(Senior)*

Record a baseline, change one variable, run a representative load test, and compare throughput, latency percentiles, allocation, GC, CPU, and memory. Roll out gradually with health checks and a rollback path.

Tuning is workload- and JDK-specific. Preserve configuration history and do not copy a flag from a different service without understanding the bottleneck it was intended to address.

### Q18. What is a good default troubleshooting order? *(Senior)*

1. Confirm the symptom and scope.
2. Check recent deployments, configuration, and traffic changes.
3. Inspect CPU, heap/RSS, GC, threads, queues, and downstream latency.
4. Capture JFR, thread dumps, and heap evidence as appropriate.
5. Form one hypothesis, test it, and measure the outcome.
6. Document the root cause, fix, and prevention actions.

This avoids random tuning and helps preserve evidence. The first visible symptom may be downstream of the actual bottleneck.

## Official references

1. [Java SE 21 tools reference](https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html)
2. [Java SE 21 JFR documentation](https://docs.oracle.com/en/java/javase/21/jfapi/index.html)
3. [Java SE 21 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
4. [Java SE 21 Native Memory Tracking](https://docs.oracle.com/en/java/javase/21/vm/native-memory-tracking.html)

## Chapter checklist

- [ ] Choose diagnostic tools based on the question being answered.
- [ ] Correlate JVM evidence with service-level latency and throughput.
- [ ] Budget total process memory, not only heap.
- [ ] Change one tuning variable at a time.
- [ ] Preserve incident evidence and document reproducible results.
