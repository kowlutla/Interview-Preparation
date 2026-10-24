# Senior Questions — API Design and Production Diagnostics

## Q&A

### Q1. What makes a REST resource identifier stable?
It should identify a resource independently of mutable display attributes. Avoid using fields that can change as primary URI identity.

### Q2. How should PATCH semantics be documented?
Define whether omitted fields mean unchanged, null means clear, and how validation/partial updates behave. Different patch formats have different semantics; do not assume clients infer them.

### Q3. What is optimistic API concurrency control?
A client supplies a version/ETag precondition, and the server rejects stale updates rather than silently overwriting concurrent changes. Choose suitable HTTP precondition/status semantics and test races.

### Q4. What is an ETag useful for?
It can support cache validation and conditional requests such as If-Match. It must be generated and interpreted consistently with the resource representation/version.

### Q5. Why are idempotency keys not just in-memory maps?
Process restarts and multiple replicas require durable coordination. Persist key, request identity, and result under a concurrency-safe uniqueness rule.

### Q6. How should API timeouts be represented?
The server should bound work and return a documented failure when possible; clients must treat a timeout as an uncertain outcome if the server may have committed before the response was lost.

### Q7. Why should request size be bounded?
Large bodies can consume memory, CPU, disk, and parser resources. Enforce limits at ingress and application layers and stream large payloads when appropriate.

### Q8. How should API versioning avoid permanent duplication?
Choose a clear compatibility strategy, define deprecation timelines, measure client use, and remove old versions only through a documented migration process.

### Q9. What is a contract test?
A test that verifies an API or message contract between producer and consumer. It catches compatibility breaks that isolated unit tests may miss.

### Q10. Why should error codes be stable?
Clients should not need to parse human-readable text or framework exception names. Stable machine-readable codes allow localized messages and implementation changes.

### Q11. What is the risk of exposing exception messages?
Messages may contain SQL, hostnames, identifiers, file paths, or secrets. Map expected errors to safe client-facing details.

### Q12. Why should a controller not own all business validation?
Controller validation handles request shape, while domain/application rules must also protect non-HTTP entry points such as messaging, scheduled jobs, and other services.

### Q13. How do you prevent mass assignment?
Bind requests to explicit DTOs rather than persistence entities with privileged fields. Allow-list mutable fields and validate state transitions in the application layer.

### Q14. What is a safe file-download design?
Authorize the resource, avoid trusting client filenames, set correct content headers, and stream large files. Prevent path traversal and avoid exposing storage paths.

### Q15. Why should HTTP client redirects be reviewed for credential leakage?
Redirects can change destinations and may cause sensitive headers to be forwarded depending on client behavior. Restrict destinations and validate redirect policy.

### Q16. How should downstream response bodies be logged?
Log safe metadata and bounded/redacted excerpts only when necessary. Avoid storing tokens, personal data, and unbounded bodies in logs.

### Q17. What does connection pool pending count indicate?
Callers are waiting to acquire connections. Correlate it with active connections, acquisition latency, transaction duration, and downstream query performance.

### Q18. What does high thread count indicate?
It can reflect expected concurrency or thread leaks, unbounded executors, blocking workloads, or pool misconfiguration. Inspect thread names/states and trends rather than relying on count alone.

### Q19. What is queue age?
The time the oldest queued task/message has waited. It often reveals user-impacting backlog earlier than a raw queue length.

### Q20. Why monitor saturation?
Saturation shows resources approaching capacity—such as executor queues, connection pools, CPU, or consumer lag—before outright errors dominate.

### Q21. How do you diagnose a deadlock?
Inspect thread dumps and database lock diagnostics, identify cycles in lock acquisition, and correlate with transaction/query logs. Avoid assuming every blocked thread is a deadlock.

### Q22. Why capture multiple thread dumps?
A single dump is a snapshot. Multiple samples show whether threads remain stuck or progress, and help distinguish persistent waits from transient work.

### Q23. What does Java Flight Recorder help with?
It records runtime events such as CPU samples, allocation, locks, and GC behavior for profiling and diagnosis with comparatively low overhead when configured appropriately.

### Q24. Why can SQL logging distort performance?
Verbose logging adds I/O and formatting overhead and can expose sensitive parameters. Use it selectively in controlled diagnostics.

### Q25. How do you validate a performance fix?
Compare representative baseline and post-change load tests, including latency percentiles, throughput, errors, resource usage, and correctness. Keep workload and environment comparable.

### Q26. What is the danger of increasing request threads?
More concurrency may increase downstream pressure, memory usage, and context switching. Size thread pools against service time and downstream capacity.

### Q27. Why is pool sizing a system-wide problem?
Each replica's pool multiplies across replicas, and databases/remote services have finite capacity. Account for rollout surge and background jobs.

### Q28. How should a service behave when its cache fails?
Define fail-open/closed behavior per operation, bound fallback concurrency, and monitor the resulting source load. Do not let a cache outage become a database outage.

### Q29. How should a service behave when logging infrastructure fails?
Application correctness should not silently depend on a remote log collector. Use appropriate local buffering/backpressure and ensure logging failure does not exhaust request threads.

### Q30. Why is an unbounded executor queue dangerous?
It can accumulate unlimited pending work, causing memory growth and stale results. Use bounded capacity and a documented rejection/backpressure strategy.

### Q31. What is a good rejection policy?
One that matches workload semantics: reject, run in caller, or persist/requeue where appropriate. Caller-runs can unexpectedly increase request latency; no policy is universally correct.

### Q32. How do you handle async cancellation?
Define whether cancellation is best-effort, propagate it to supported upstream work, and ensure resource cleanup. A caller timing out does not automatically cancel every downstream side effect.

### Q33. Why is task durability a separate requirement from async execution?
Executor tasks generally live in process memory. Durable queues/jobs preserve work across process restarts and allow recovery/replay.

### Q34. What is a good alert?
A signal tied to user impact or a meaningful saturation condition, with a threshold and window that avoids noise. Alerts should point to a runbook and an owner.

### Q35. Why should dashboards show deployment version?
It allows responders to correlate behavior changes with canaries, rollouts, and dependency upgrades.

### Q36. What is a useful deployment smoke test?
A bounded set of checks that verify startup, critical route behavior, auth, database connectivity, downstream integration, and observability in the deployed environment.

### Q37. Why test failure paths before release?
Most reliability defects appear under timeouts, partial failures, retries, concurrency, or shutdown—not the happy path. Failure injection validates recovery assumptions.

### Q38. What is chaos testing's limitation?
It only tests scenarios and scope that are exercised; a successful experiment does not prove the system is resilient to every failure. Start with controlled, observable experiments.

### Q39. How should an incident timeline be recorded?
Record impact onset, detection, mitigation, changes, recovery, and evidence with timestamps. A factual timeline supports learning without blame.

### Q40. What should a post-incident action item contain?
An owner, measurable outcome, due date, and verification method. “Be more careful” is not an actionable reliability improvement.

### Q41. How do you choose what to fix first during an incident?
Prioritize restoring user impact safely, preventing amplification, and preserving data correctness. Avoid risky broad changes before narrowing the fault domain.

### Q42. What is a safe emergency configuration change?
A narrowly scoped, reversible change with known side effects, audited approval, and telemetry to verify impact. Avoid unreviewed changes to authentication, authorization, or transaction semantics.

### Q43. Why does a successful health probe not replace synthetic monitoring?
A probe may exercise only a lightweight internal path. Synthetic checks can validate critical end-to-end journeys and user-visible outcomes.

### Q44. How should dependency health affect API responses?
Critical dependency failure may require a clear unavailable response; noncritical dependencies can degrade gracefully if the domain contract allows it. Avoid fabricating success.

### Q45. What is the role of a runbook?
It provides tested diagnostic and mitigation steps, safety cautions, and escalation paths for recurring operational problems.

### Q46. How should support bundles be handled?
Limit included data, redact secrets, restrict access, and set retention. Dumps and logs can contain sensitive production information.

### Q47. What does version drift cause?
Different instances may run different dependency behavior or configuration, complicating diagnosis and compatibility. Track image digest and resolved dependency versions.

### Q48. How do you validate dependency vulnerability remediation?
Identify the affected artifact and transitive path, upgrade to a compatible fixed version, run regression tests, and verify the deployed artifact contains the remediation.

### Q49. Why is a reproducible build important?
It makes dependency and artifact changes auditable and helps distinguish source changes from environment drift. Pin toolchains and avoid mutable dependency inputs where possible.

### Q50. How do you know the handbook itself is technically reliable?
Every claim should be tied to an authoritative reference or verified example, version-dependent statements should name their baseline, and examples should be reviewed for correctness, duplication, and consistency.
