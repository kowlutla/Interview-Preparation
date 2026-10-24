# Advanced Questions — Clients, Reactive, and Observability

## Q&A

### Q1. What is a connection timeout?
It bounds establishing a connection to a remote endpoint. It is distinct from waiting for a response or an overall operation deadline.

### Q2. Why configure a response timeout?
A server can accept a connection but then respond slowly. Without bounded response handling, resources can remain occupied and propagate latency.

### Q3. What is a deadline budget?
The maximum time allowed for the complete operation, including downstream calls, retries, and response processing. Each nested timeout should fit within it.

### Q4. Why can retries amplify an outage?
Multiple layers may retry the same logical request, multiplying attempts precisely when a dependency is overloaded. Bound attempts and coordinate policies.

### Q5. What is jitter in retry backoff?
Random variation added to retry delays to reduce synchronized retry bursts from many clients.

### Q6. What is a bulkhead?
A resource isolation mechanism that limits how much one workload or dependency can consume, such as separate bounded executors or concurrency limits.

### Q7. Does a circuit breaker replace bulkheads?
No. Circuit breakers respond to observed failures; bulkheads isolate resource consumption. They can complement one another.

### Q8. What is a safe fallback?
A fallback that preserves the domain contract, such as omitting a noncritical recommendation. It must not fabricate success for payment, authorization, or critical writes.

### Q9. What is the difference between RestClient and WebClient?
RestClient is synchronous and fluent; WebClient supports non-blocking reactive I/O. Choose based on runtime baseline and end-to-end workload.

### Q10. Is RestTemplate removed?
It remains in existing applications and supported framework lines, but current client selection should follow the chosen version's official guidance rather than assuming new APIs are available everywhere.

### Q11. Why is backpressure not a universal memory limit?
Backpressure coordinates demand between compatible publishers and subscribers, but buffering operators, hot sources, adapters to blocking APIs, and external queues may still accumulate work under their own policies. Inspect each buffering boundary, configure explicit limits where available, and define behavior for overflow or cancellation. Backpressure is a flow-control mechanism, not a guarantee that every queue is bounded.

### Q12. Does Flux execute work in parallel?
Not inherently. Reactive composition describes signals; concurrency requires explicit operators/schedulers or concurrent upstream sources.

### Q13. What does subscribeOn affect?
It influences where subscription/source work is scheduled for supported publishers. Operator placement and source behavior matter; it is not a universal thread-switch for every operation.

### Q14. What does publishOn affect?
It shifts downstream signal processing to a scheduler from that point in the pipeline. It does not make preceding blocking work non-blocking.

### Q15. Why avoid block() in WebFlux request paths?
It can block an event-loop worker and reduce concurrency. Use reactive composition or isolate unavoidable blocking operations appropriately.

### Q16. What is cancellation propagation?
A cancellation signal asks upstream publishers to stop producing work. Actual resource cancellation depends on the publisher/client and integration.

### Q17. Why can reactive context differ from ThreadLocal?
Reactive pipelines may hop threads, so thread-bound values do not automatically follow each signal. Reactor Context and supported integrations carry context across operators.

### Q18. What is StepVerifier used for?
It tests reactive publishers by asserting emitted signals, errors, completion, and demand/time behavior.

### Q19. Why are unbounded reactive buffers risky?
They can accumulate data faster than it is processed, causing memory pressure. Backpressure, bounded buffering, and explicit overflow policies are important.

### Q20. What is Micrometer's role?
It provides instrumentation abstractions for metrics and integrations with monitoring backends.

### Q21. What is a counter?
A monotonically increasing measurement of occurrences, such as failed requests. Use a timer for duration and distribution information.

### Q22. What is a histogram useful for?
It provides distribution buckets that can support percentile approximation and latency SLO analysis in supported backends.

### Q23. Why avoid high-cardinality metric tags?
Each unique tag combination creates additional time series, increasing storage and query cost. Keep labels bounded and move unique IDs to logs/traces.

### Q24. What are RED metrics?
Rate, errors, and duration are a practical set for monitoring request-oriented services.

### Q25. What are USE metrics?
Utilization, saturation, and errors help reason about resource bottlenecks.

### Q26. Why do logs need correlation IDs?
They help connect related events across layers/services. Use supported trace context when available and avoid trusting arbitrary IDs without validation.

### Q27. What does distributed tracing show?
A request's spans across services and time spent in each operation. It helps localize latency but may be sampled and does not guarantee every request is recorded.

### Q28. Why not log full HTTP payloads by default?
Payloads may contain credentials, personal data, large content, and secrets. Prefer safe metadata and carefully controlled redaction.

### Q29. What is the difference between readiness and health?
Health is a broad status concept; readiness specifically determines whether an instance should receive traffic. Endpoint semantics must be configured to match operational needs.

### Q30. Why should liveness not fail on every downstream outage?
All instances could restart repeatedly while a shared dependency remains down, worsening recovery. Liveness should indicate process progress, while readiness can represent traffic eligibility.

### Q31. What is a custom HealthIndicator for?
It adds application- or dependency-specific status. Checks should be bounded, low-cost, and not expose sensitive details.

### Q32. Why can a health check cause cascading load?
Frequent probes across many replicas can trigger expensive database/network calls during an outage. Keep checks bounded and avoid full business workflows.

### Q33. What is startup telemetry useful for?
It identifies slow startup phases and bean initialization bottlenecks, enabling measurement-driven optimization.

### Q34. How do you diagnose high CPU?
Use CPU profiles/JFR, thread dumps, request metrics, and allocation data to identify hot loops, serialization, excessive logging, or contention. Do not infer the cause from CPU alone.

### Q35. How do you diagnose high GC pressure?
Inspect allocation rate, GC logs/metrics, object lifetime, heap occupancy after GC, and allocation profiles. Large heap changes may mask excessive allocation.

### Q36. What is a heap dump dominator tree?
A view of objects retaining large portions of the heap, useful for finding retention roots and oversized collections.

### Q37. Why inspect native memory?
Direct buffers, thread stacks, metaspace, and native libraries consume memory outside the Java heap and can cause container OOM.

### Q38. What is a thread dump deadlock indicator?
A JVM may report deadlocked threads; otherwise inspect lock ownership and cycles across multiple dumps. Not all stalls are deadlocks—some are slow I/O or resource starvation.

### Q39. What is connection acquisition latency?
Time spent waiting to borrow a connection from a pool. It can reveal pool saturation even when database query time itself is low.

### Q40. Why measure p95/p99 instead of averages only?
Averages hide long-tail requests that often dominate user experience and timeout failures.

### Q41. What is a useful load test?
One that resembles production payloads, concurrency, dependency latency, data distribution, and failure modes, with repeatable measurement.

### Q42. Why not optimize based on local microbenchmarks alone?
Local benchmarks may omit network, contention, GC, database, and deployment effects. Use them for narrow questions, then validate end-to-end.

### Q43. What is a canary rollout?
A small subset of traffic is routed to a new version to compare operational behavior before broader rollout.

### Q44. Which signals should gate a canary?
User-visible error rate, latency percentiles, saturation, and business-specific correctness metrics, with thresholds and observation windows defined in advance.

### Q45. What is the purpose of an incident runbook?
It gives responders validated diagnostic and mitigation steps, including safety checks and rollback paths, reducing improvisation during outages.
