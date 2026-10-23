# Real-World Java Production Scenarios (1–50)

These scenarios train the reasoning expected from a senior Java backend engineer. For each case, explain symptoms, hypotheses, evidence to collect, immediate mitigation, root-cause fix, and prevention. Avoid proposing a restart or JVM flag as the only answer.

## Collections and data correctness

### Scenario 1. HashMap lookup returns null for a key that was inserted

**Likely causes:** Key fields used by equals/hashCode changed after insertion; inconsistent equality/hash contract; a different class-loader identity; or the key was never inserted as assumed.

**Investigate:** Compare key state/hash at insertion and lookup, inspect equals/hashCode, and verify containsKey versus null-value semantics.

**Fix:** Use stable immutable keys and test equality contracts.

### Scenario 2. HashSet contains two apparently identical domain objects

**Likely causes:** equals/hashCode are not overridden consistently, or business uniqueness differs from object equality.

**Investigate:** Compare distinct instances, hashes, and the domain key.

**Fix:** Implement value equality only when it matches the domain, or use a Set/Map keyed by an explicit immutable business ID.

### Scenario 3. A test fails intermittently because map output order changes

**Likely cause:** Business logic or tests rely on unspecified HashMap iteration order.

**Investigate:** Find assertions, serialization, or logs that depend on traversal order.

**Fix:** Use LinkedHashMap for insertion order or sort explicitly for deterministic output.

### Scenario 4. ConcurrentModificationException occurs in one thread

**Likely cause:** Structural mutation of a collection while its iterator is active.

**Investigate:** Trace helpers/callbacks that add/remove elements during traversal.

**Fix:** Use Iterator.remove, removeIf, or a two-phase collect-then-mutate approach.

### Scenario 5. A tiny subList keeps a large list in memory

**Likely cause:** The subList view retains its backing list.

**Investigate:** Heap dump and reference path from the long-lived view.

**Fix:** Create an independent copy when the subset must outlive the source.

### Scenario 6. A cache grows until heap exhaustion

**Likely cause:** A plain map is being used as an unbounded cache with no expiry/eviction.

**Investigate:** Entry count, key cardinality, retained-size paths, and insertion lifecycle.

**Fix:** Set maximum size/TTL, define eviction semantics, and monitor hit/miss/eviction metrics.

### Scenario 7. TreeSet silently drops distinct objects

**Likely cause:** Comparator returns zero for objects that equals considers different.

**Investigate:** Comparator results for distinct values and intended uniqueness.

**Fix:** Make comparator consistent with equals or document the intended comparison-based equivalence and use a different structure if needed.

### Scenario 8. ArrayList writes are lost under load

**Likely cause:** Unsynchronized concurrent mutation of a non-thread-safe list.

**Investigate:** Identify all writers and compound operations; reproduce under concurrency stress.

**Fix:** Use confinement, locking, a concurrent collection, or immutable snapshots appropriate to the contract.

### Scenario 9. Collection iteration shows stale values

**Likely cause:** Unsafe publication or concurrent mutation without a defined visibility protocol.

**Investigate:** Shared references, happens-before edges, and whether the collection is thread-safe.

**Fix:** Publish safely and use the correct synchronization/concurrent collection semantics.

### Scenario 10. A ConcurrentHashMap-based cache still duplicates expensive loads

**Likely cause:** Check-then-act code or separate get and put calls permit duplicate work.

**Investigate:** Loading sequence and races for the same key.

**Fix:** Use computeIfAbsent or a deliberate future-valued per-key loading protocol, keeping mapping work short and failure cleanup explicit.

## Concurrency and task execution

### Scenario 11. A service has high thread count and low CPU

**Likely causes:** Blocking I/O, connection-pool waits, lock contention, or unbounded platform-thread creation.

**Investigate:** Thread dumps, thread states, executor queues, connection-pool wait time, and downstream latency.

**Fix:** Bound concurrency, remove blocking from critical paths, and use virtual threads or async APIs only where the workload and dependencies support them.

### Scenario 12. All executor workers are waiting for tasks submitted to the same executor

**Likely cause:** Thread-starvation deadlock from nested blocking submissions.

**Investigate:** Dump worker stacks and inspect Future.get/join dependencies.

**Fix:** Compose asynchronously, separate execution domains, or avoid waiting for queued tasks from the same constrained pool.

### Scenario 13. The executor queue grows but CPU remains low

**Likely causes:** Workers blocked on network/database I/O, lock contention, slow downstreams, or too few workers for a blocking workload.

**Investigate:** Queue wait time, task duration, thread dumps, and downstream pool metrics.

**Fix:** Fix the bottleneck, bound the queue, and define backpressure/rejection policy rather than simply increasing max threads.

### Scenario 14. Tasks disappear without visible exceptions

**Likely cause:** submit returned a Future whose failure was never observed, or a rejection/discard policy silently dropped work.

**Investigate:** Futures, rejection handler, completion/failure metrics, and executor shutdown state.

**Fix:** Observe task outcomes and use explicit rejection/error reporting for required work.

### Scenario 15. Application shutdown hangs indefinitely

**Likely causes:** Non-daemon threads, tasks ignoring interruption, blocked I/O, or executor shutdown waiting without a deadline.

**Investigate:** Thread dumps and shutdown lifecycle logs.

**Fix:** Implement bounded graceful shutdown, cooperative cancellation, resource closure, and interruption restoration.

### Scenario 16. A shutdown flag is sometimes ignored

**Likely cause:** A non-volatile shared flag or unsynchronized access.

**Investigate:** Shared-state access paths and JMM happens-before relationships.

**Fix:** Use volatile or locking for visibility, plus a wake-up/cancellation mechanism if the worker is blocked.

### Scenario 17. A volatile counter undercounts requests

**Likely cause:** Increment is a read-modify-write operation and is not atomic.

**Investigate:** Identify all concurrent increments and compare with an independent request count.

**Fix:** Use AtomicLong or LongAdder based on whether exact atomic reads or high-contention metrics are required.

### Scenario 18. Deadlock appears after adding a second lock

**Likely cause:** Different code paths acquire locks in opposite order.

**Investigate:** Thread dump lock ownership and build a wait-for graph.

**Fix:** Define global lock ordering, reduce nested locks, and avoid external calls inside critical sections.

### Scenario 19. A CompletableFuture chain stalls

**Likely causes:** A continuation blocks a constrained executor, an exception is unobserved, a dependency never completes, or code calls join/get in a worker needed for progress.

**Investigate:** Stage graph, executor saturation, timeouts, and exceptional completion.

**Fix:** Use non-blocking composition, explicit executors for blocking work, deadlines, and centralized failure handling.

### Scenario 20. A cancelled task still performs a remote write

**Likely cause:** Cancellation/interruption is cooperative and the remote operation already committed.

**Investigate:** Request/response timing, remote idempotency, and cancellation propagation.

**Fix:** Use idempotency keys, server-side deduplication, deadlines, and reconciliation for uncertain outcomes.

### Scenario 21. Virtual threads improve thread count but not latency

**Likely causes:** Database or remote service saturation, connection-pool bottleneck, CPU work, or excessive downstream concurrency.

**Investigate:** Dependency wait time, pool saturation, CPU, and live virtual-thread count.

**Fix:** Bound scarce resources and optimize the actual bottleneck; virtual threads do not increase database capacity.

### Scenario 22. Virtual-thread workload suffers under synchronized blocking sections

**Likely cause:** In the relevant Java 21 runtime circumstances, blocking while pinned can reduce carrier availability.

**Investigate:** JFR and virtual-thread pinning diagnostics, then identify blocking operations inside synchronized regions.

**Fix:** Keep synchronized sections short and move blocking I/O outside where possible; do not mechanically replace all synchronized blocks.

### Scenario 23. ConcurrentHashMap cache returns stale or inconsistent aggregate values

**Likely cause:** Individual map operations are thread-safe but a multi-key invariant is not atomic.

**Investigate:** State transition spanning keys and readers' consistency requirements.

**Fix:** Use an explicit lock/transaction, immutable snapshot, or redesigned per-key invariant.

### Scenario 24. A ThreadLocal request context leaks across requests

**Likely cause:** A pooled worker retains a value after the request completes.

**Investigate:** ThreadLocal set/remove paths, including exceptions and cancellation.

**Fix:** Clear in finally and prefer explicit context propagation where appropriate.

### Scenario 25. A scheduled task stops running after one failure

**Likely cause:** An exception escaped a periodic task and the scheduler suppressed subsequent executions according to its contract.

**Investigate:** Capture the task exception and scheduler configuration.

**Fix:** Handle expected failures inside the task, surface unexpected failures through metrics/alerts, and define whether retry is safe.

## JVM, memory and performance

### Scenario 26. Heap usage rises steadily after every deployment

**Likely causes:** Class-loader leak, static registry retention, listener leak, or cache growth.

**Investigate:** Compare heap dumps, class-loader counts, dominator trees, and retained paths across deployments.

**Fix:** Clean global registrations and bound long-lived caches; do not only increase heap.

### Scenario 27. Heap is healthy but the container is OOM-killed

**Likely causes:** Native memory, direct buffers, thread stacks, Metaspace, code cache, native libraries, or a container limit below total process RSS.

**Investigate:** Cgroup/kernel OOM events, RSS, NMT, thread count, direct-buffer usage, and heap configuration.

**Fix:** Budget total process memory and set heap maximum below the container limit with measured headroom.

### Scenario 28. OutOfMemoryError: Metaspace occurs after hot redeploys

**Likely causes:** Class loaders cannot be unloaded because threads, ThreadLocals, static registries, or callbacks retain them.

**Investigate:** Loader counts, class-loading metrics, and heap paths to old loaders.

**Fix:** Release global references and thread context, and verify unloading rather than only raising Metaspace maximum.

### Scenario 29. GC pauses increase after a new release

**Likely causes:** Higher allocation rate, larger live set, changed workload, new object graph retention, or runtime/collector configuration changes.

**Investigate:** Compare GC logs, JFR allocation profiles, heap occupancy, and request throughput against the baseline.

**Fix:** Identify allocation/retention changes first, then tune collector or heap based on measured goals.

### Scenario 30. High CPU occurs after warmup

**Likely causes:** Hot algorithm, spin loop, excessive retries, lock contention, GC/JIT activity, or increased request load.

**Investigate:** CPU profiles, hot threads, JFR, allocation, and traffic changes.

**Fix:** Optimize the dominant stack and validate with a reproducible benchmark; do not blindly add threads.

### Scenario 31. Latency spikes correlate with GC but not every GC causes a spike

**Likely causes:** Some collections overlap with high-load requests or allocation bursts; correlation alone does not prove causation.

**Investigate:** Align request traces, GC pause timestamps, CPU, and queueing data.

**Fix:** Quantify contribution and address the dominant factor, not merely the nearest GC event.

### Scenario 32. Java heap OOM occurs despite low average heap utilization

**Likely causes:** Allocation burst, very large object, fragmentation/collector-specific constraints, sudden concurrency, or misleading average metrics.

**Investigate:** Peak heap, GC logs, allocation profile, heap dump, and request-size distribution.

**Fix:** Bound input/batch sizes, reduce peaks, and configure memory from peak behavior rather than averages.

### Scenario 33. CPU profile shows String/regex methods dominating

**Likely causes:** Repeated regex compilation, excessive string copying, poorly bounded patterns, or repeated parsing.

**Investigate:** Call counts, allocation profile, input sizes, and regex behavior.

**Fix:** Reuse Pattern where appropriate, simplify parsing, use builders for repeated assembly, and defend against pathological inputs.

### Scenario 34. Startup is slow but steady-state throughput is good

**Likely causes:** Classpath scanning, reflection, class loading, JIT warmup, static initialization, or slow dependency startup.

**Investigate:** Startup timing, class-load logs, JFR, and framework initialization phases.

**Fix:** Reduce unnecessary scanning/initialization, avoid external I/O in static initializers, and optimize startup separately from steady-state performance.

### Scenario 35. A new JDK changes memory footprint

**Likely causes:** Collector/ergonomics changes, runtime metadata, different allocation/JIT behavior, or changed application/library behavior.

**Investigate:** Same workload, same limits, exact JDK build, GC logs, NMT, and allocation profiles.

**Fix:** Establish a comparable baseline and tune only after locating the category responsible.

## I/O, transactions and security

### Scenario 36. A JDBC pool is exhausted while the database looks underutilized

**Likely causes:** Connections are leaked, transactions are held too long, callers wait while holding connections, or connection acquisition/use patterns are inefficient.

**Investigate:** Active/idle/pending pool metrics, transaction duration, thread dumps, and connection leak diagnostics.

**Fix:** Close resources, shorten transactions, avoid remote calls inside transactions, and size the pool to database capacity.

### Scenario 37. A payment retry creates duplicate charges

**Likely cause:** A timeout was interpreted as proof that the first request failed, although the provider may have committed the charge.

**Investigate:** Request IDs, provider idempotency support, response loss timing, and retry policy.

**Fix:** Use idempotency keys, persist operation state, and reconcile unknown outcomes instead of blindly repeating the side effect.

### Scenario 38. Database writes partially succeed in a batch

**Likely causes:** Batch execution can return partial counts or fail after some statements, depending on driver/database behavior and transaction configuration.

**Investigate:** Auto-commit, transaction boundaries, BatchUpdateException details, and driver semantics.

**Fix:** Define rollback/commit policy and test partial failures against the actual database driver.

### Scenario 39. User input causes SQL injection

**Likely cause:** Input is concatenated into SQL syntax.

**Investigate:** Query construction and dynamic identifier handling.

**Fix:** Use parameterized statements; allowlist identifiers that cannot be bound as values; use least-privilege database credentials.

### Scenario 40. An HTTP request hangs for minutes

**Likely causes:** Missing connect/request deadlines, blocked body reads, retry loops, DNS delays, or downstream pool waits.

**Investigate:** Per-stage timing, thread dumps, connection pool metrics, and trace spans.

**Fix:** Define end-to-end deadlines, bound retries, close response bodies, and propagate cancellation.

### Scenario 41. A TLS connection fails only in production

**Likely causes:** Trust-store differences, certificate chain/hostname mismatch, proxy interception, or outdated runtime certificates.

**Investigate:** Exact JDK build, trust configuration, certificate chain, and handshake diagnostics.

**Fix:** Correct certificate/trust configuration; never disable hostname or certificate validation as a workaround.

### Scenario 42. A path validation check is bypassed

**Likely causes:** Normalization differences, symbolic links, alternate encodings, or a time-of-check/time-of-use race.

**Investigate:** Resolve the path against a trusted root and examine the actual filesystem target and symlink policy.

**Fix:** Use path-aware validation, least privilege, and race-aware operations rather than string-prefix checks.

### Scenario 43. A public API leaks internal exception details

**Likely cause:** Stack traces, SQL errors, paths, or dependency messages are returned directly to clients.

**Investigate:** Error mapping and logging behavior.

**Fix:** Return a stable safe error schema with a correlation ID; retain sanitized diagnostic context internally.

### Scenario 44. Java deserialization causes a security incident

**Likely cause:** Untrusted native serialized data reaches ObjectInputStream with a dangerous classpath.

**Investigate:** Entry point, payload source, reachable classes, and dependency versions.

**Fix:** Remove native deserialization for untrusted data where possible; otherwise apply strict ObjectInputFilter allowlists and limits, and patch dependencies.

### Scenario 45. A file-processing job exhausts heap

**Likely cause:** Whole-file reads, unbounded batches, or accumulating results in memory.

**Investigate:** Input size, allocation profile, buffering, and retained collections.

**Fix:** Stream records, bound batches and queues, apply backpressure, and define partial-failure recovery.

## Release, API and operational scenarios

### Scenario 46. Code compiles on JDK 21 but fails on Java 17

**Likely cause:** The code uses newer language features or APIs unavailable on the runtime baseline.

**Investigate:** Compile with --release 17, inspect class-file/API dependencies, and verify deployment JDK.

**Fix:** Align source, API, bytecode target, and runtime compatibility; do not rely on the developer's default JDK.

### Scenario 47. A reflection-based framework fails after modularization

**Likely cause:** JPMS package is not open to the framework module or a dependency relies on internal JDK APIs.

**Investigate:** InaccessibleObjectException, module descriptor, and supported framework version.

**Fix:** Prefer supported public APIs or add narrowly scoped opens directives; avoid broad internal-access flags as permanent architecture.

### Scenario 48. A record exposes secrets in logs

**Likely cause:** Generated toString includes record components.

**Investigate:** Logging of entire records and nested DTOs.

**Fix:** Avoid logging sensitive objects wholesale; define safe redacted representations and structured fields.

### Scenario 49. A preview API fails in CI

**Likely cause:** Compiler/runtime preview flags or JDK versions differ.

**Investigate:** Build toolchain, compile flags, runtime flags, and exact JDK release.

**Fix:** Pin the JDK and flags, isolate preview usage, or migrate to the finalized feature when available.

### Scenario 50. Performance regresses after a “small” refactor

**Likely causes:** Changed allocation, call-site profile, lock scope, query count, object retention, or hidden ordering/algorithmic behavior.

**Investigate:** Compare profiles, JFR, allocation, latency percentiles, database queries, and benchmark the changed path.

**Fix:** Reproduce the regression, identify the changed resource cost, and add a regression test/benchmark or metric to prevent recurrence.

## Scenario response template

For any incident, answer in this order: **impact and scope → immediate safety mitigation → evidence → ranked hypotheses → root cause → corrective change → regression test → monitoring/prevention**. Avoid claiming root cause until evidence distinguishes it from competing explanations.
