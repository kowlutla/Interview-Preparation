# JVM and Memory Incident Scenarios

These scenarios extend the initial production scenario bank with focused JVM diagnosis. For each incident, identify the exact symptom, gather evidence, distinguish competing causes, and define a measurable fix.

### Scenario 1. Heap grows but post-GC live set remains stable

**Interpretation:** This may be normal allocation churn rather than a leak.

**Evidence:** GC logs, allocation profiling, live-set trend, request rate.

**Action:** Find allocation hotspots and assess GC cost before changing heap size.

### Scenario 2. Post-GC live set grows with stable traffic

**Interpretation:** Likely unintended retention, cache growth, or changed object lifecycle.

**Evidence:** Comparable heap dumps, dominator tree, retained size, root paths.

**Action:** Identify the retaining reference and add a lifecycle/bounds regression test.

### Scenario 3. Metaspace grows after every redeploy

**Interpretation:** Old class loaders or generated classes may remain reachable.

**Evidence:** Class loading/unloading counters, loader count, heap paths to loaders.

**Action:** Remove stale registrations, threads, ThreadLocals, and global references.

### Scenario 4. Direct memory exhaustion with free heap

**Interpretation:** Direct buffers/native allocations can exceed their budget independently of heap.

**Evidence:** Direct buffer pool metrics, NMT, RSS, cgroup events.

**Action:** Bound buffers, fix retention, and budget native memory.

### Scenario 5. Unable to create native thread

**Interpretation:** Thread/OS/native-memory limits may be reached.

**Evidence:** Thread count, stack configuration, executor sizes, process limits, container memory.

**Action:** Eliminate unbounded thread creation and bound task concurrency.

### Scenario 6. Long GC pause occurs after a traffic burst

**Interpretation:** Allocation spike, live-set growth, promotion pressure, or collector behavior may be involved.

**Evidence:** GC event details, allocation rate, pause timeline, request rate.

**Action:** Reduce burst memory or tune only after confirming the collector is the bottleneck.

### Scenario 7. CPU is high during garbage collection

**Interpretation:** GC may be consuming significant CPU, but application CPU can also be high concurrently.

**Evidence:** JFR, GC logs, CPU profile, allocation rate.

**Action:** Separate GC CPU from application hot paths and test one change at a time.

### Scenario 8. OOM occurs only in containers

**Interpretation:** The container limit may be lower than total process memory needs.

**Evidence:** cgroup memory, RSS, heap maximum, Metaspace, stacks, direct buffers.

**Action:** Set heap based on total process budget and validate under peak load.

### Scenario 9. CPU spikes after JDK upgrade

**Interpretation:** Runtime flags, JIT profile, GC, dependencies, or changed application behavior may differ.

**Evidence:** Exact build, flags, JFR, CPU profile, GC comparison.

**Action:** Reproduce on both JDKs with the same workload and isolate the change.

### Scenario 10. Startup uses much more memory than expected

**Interpretation:** Framework scanning, class loading, caches, or eager initialization may allocate heavily.

**Evidence:** Startup JFR, class-loading logs, heap and native-memory snapshots.

**Action:** Separate startup from steady state and remove unnecessary eager work.

### Scenario 11. Thread dump shows many WAITING threads

**Interpretation:** They may be healthy workers waiting on queues or may be stalled behind a dependency.

**Evidence:** Stack frames, wait target, queue depth, completion metrics.

**Action:** Determine whether the wait is expected and whether producers or consumers are progressing.

### Scenario 12. JIT code cache is near capacity

**Interpretation:** Compiled code storage may constrain compilation and steady-state performance.

**Evidence:** Code-cache metrics and compiler logs.

**Action:** Confirm the code cache is the bottleneck before changing capacity; inspect compilation activity and workload churn.

### Scenario 13. Class unloading stops

**Interpretation:** Class loaders may remain reachable or unloading conditions may not be met.

**Evidence:** Loader references, threads, static caches, class unloading logs.

**Action:** Remove retaining references and validate loader lifecycle.

### Scenario 14. GC logs show frequent collections but low pause time

**Interpretation:** Total GC CPU and frequency may still reduce throughput even when pauses are short.

**Evidence:** Total GC time, allocation rate, CPU, throughput, live set.

**Action:** Optimize allocation or collector settings based on throughput and latency objectives together.

### Scenario 15. Heap dump capture itself threatens service health

**Interpretation:** Dumps can pause the process, consume disk, and expose sensitive data.

**Evidence:** Heap size, disk space, operational runbook, data sensitivity.

**Action:** Plan capture safely, restrict access, ensure storage capacity, and prefer lower-impact JFR/metrics first when suitable.

## Incident template

Record: exact JDK/build and flags, container limits, symptom window, traffic shape, heap/live-set trend, GC events, CPU, thread count, native memory, evidence-backed root cause, fix, regression test, and monitoring.
