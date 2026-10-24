# Senior Questions — Architecture and Engineering Judgment

## Q&A

### Q1. How do you decide whether a concern belongs in a filter, interceptor, or AOP aspect?
Choose the layer that owns the concern: servlet-wide request/response processing belongs in filters, MVC handler lifecycle in interceptors, and cross-cutting method behavior on Spring-managed services in AOP. Avoid duplicating policy across layers without a clear reason.

### Q2. How do you prevent a Spring service from becoming a god object?
Separate cohesive responsibilities, keep dependencies purposeful, and move orchestration only where it improves domain clarity. A large constructor is a signal to review responsibilities, not automatically a reason to create interfaces.

### Q3. When is a modular monolith preferable to microservices?
When domain boundaries are still evolving, independent deployment is not needed, or operational capacity is limited. A modular monolith can preserve clear module contracts without distributed failure and deployment overhead.

### Q4. What is the cost of microservice boundaries?
Network failure, latency, distributed tracing needs, data ownership, eventual consistency, deployment coordination, and operational burden. Split services for independent capabilities and ownership, not just package organization.

### Q5. How do you define a transaction boundary?
Around a business operation that needs atomic local state changes. Keep it short and avoid remote I/O inside it; model cross-resource coordination explicitly.

### Q6. How do you decide between synchronous HTTP and messaging?
Use synchronous calls when the caller needs an immediate answer and coupling is acceptable. Use messaging for asynchronous decoupling, buffering, and durable workflows, accepting eventual consistency and operational complexity.

### Q7. What makes an API backward compatible?
Existing clients continue to work with the new behavior and schema. Avoid removing/renaming fields or changing semantics without a versioning/migration plan; additive changes can still break strict clients.

### Q8. When should an API be versioned?
When incompatible contract changes cannot be introduced compatibly. Choose a documented strategy and support overlap/migration; versioning does not excuse poor compatibility discipline.

### Q9. How do you protect internal service APIs?
Use service identity, network boundaries, token validation, least-privilege authorization, and careful header propagation. Network location alone is not identity.

### Q10. How should a service handle partial failure in a fan-out request?
Set a total deadline, classify dependencies by criticality, bound concurrency, and define whether partial results are valid. Do not silently turn missing critical data into success.

### Q11. What is the danger of synchronous fan-out?
Latency can accumulate across sequential calls, and one slow dependency can hold request resources. Parallel fan-out can reduce latency but increases concurrency and failure complexity.

### Q12. How do you design idempotent APIs?
Define a stable operation identity, persist the outcome or deduplication key, enforce uniqueness under concurrency, and return a consistent result for repeats.

### Q13. What is a saga?
A sequence of local transactions coordinated across services with compensating actions or forward recovery. It is not a global ACID transaction, and compensation may not perfectly reverse external effects.

### Q14. When is an outbox not enough?
It solves reliable publication intent, but not all multi-step business workflows, compensation, ordering, or consumer-side idempotency. Additional orchestration may be needed.

### Q15. How do you choose between orchestration and choreography?
Orchestration centralizes workflow state and decisions; choreography uses events and distributed reactions. Consider visibility, coupling, complexity, ownership, and how operators will diagnose a stuck workflow.

### Q16. How should API rate limiting fail when its backing store is down?
The policy must be explicit: fail open or closed based on abuse risk and availability needs. Document the trade-off, instrument limiter errors, and avoid an accidental unlimited bypass.

### Q17. How do you keep retries from creating a retry storm?
Use bounded attempts, exponential backoff with jitter, one clear retry owner where possible, a total deadline, and concurrency limits. Observe retry volume and disable unsafe retries during overload.

### Q18. What is load shedding?
Rejecting or deferring work when the system cannot safely handle more load. It preserves capacity for higher-priority work and prevents queues from growing without bound.

### Q19. How do you set an SLO for a Spring API?
Define a user-visible indicator such as successful requests within a latency threshold over a window. Ensure instrumentation captures failures and latency accurately and align alerts with user impact.

### Q20. What is an error budget?
The amount of unreliability permitted by an SLO over a period. It helps balance feature delivery against reliability work and can inform rollout policy.

### Q21. How do you use feature flags safely?
Keep flag ownership, default behavior, expiry, and rollback semantics explicit. Avoid accumulating permanent branching logic and do not use flags as a substitute for authorization.

### Q22. What is a safe schema evolution strategy?
Use backward-compatible additive changes first, deploy code that tolerates both forms, migrate data, and remove old schema only after all old versions are gone.

### Q23. Why are message schemas contracts?
Producers and consumers can deploy independently. Use compatibility rules, stable event identity, schema evolution policy, and consumer tests to prevent breaking changes.

### Q24. How do you handle poison messages?
Classify the failure, bound retries, isolate the message for inspection, preserve diagnostic metadata, and provide controlled replay after the cause is fixed.

### Q25. How do you ensure scheduled work is recoverable?
Persist progress, use idempotent chunks, record outcomes, and support restart/resume. A scheduler only triggers work; it does not provide durable workflow semantics by itself.

### Q26. How should batch size be chosen?
Balance throughput against memory, transaction duration, lock time, retry cost, and downstream capacity. Measure representative workloads and define partial-batch failure behavior.

### Q27. What is the risk of one transaction per huge batch?
It can hold locks and connections too long, grow transaction logs, and make retries expensive. Chunking can improve recovery but changes atomicity semantics.

### Q28. How do you keep caches from leaking tenant data?
Include tenant identity in keys and enforce authorization before returning cached content. Test collisions and administrative access paths.

### Q29. How do you decide cache freshness?
Define the business tolerance for stale data, identify write paths, and choose invalidation/TTL based on the consistency contract. TTL alone is not a strong invalidation guarantee.

### Q30. What should be cached?
Expensive, frequently reused data with acceptable freshness and a clear key/invalidation strategy. Avoid caching sensitive or highly personalized data without correct scoping.

### Q31. How do you design a safe fallback?
It must preserve domain invariants, be clearly observable, and have an expiry/recovery path. Never convert a critical failure into fabricated success.

### Q32. How do you decide whether to fail open or closed?
Evaluate security risk, correctness, availability impact, and the consequences of accepting versus rejecting requests. Record the decision and test dependency failure behavior.

### Q33. How do you manage secrets in local development?
Use approved secret tooling or environment-specific local configuration outside source control, with least privilege and rotation. Never commit realistic production credentials.

### Q34. What is configuration drift?
Different instances/environments run with unintended settings. Reduce it through declarative configuration, versioned manifests, validation, and inspection of effective non-secret values.

### Q35. How do you make a migration observable?
Emit migration status and duration, capture clear failure diagnostics, and alert on incomplete rollout. Avoid exposing sensitive schema/data details in public endpoints.

### Q36. How do you choose a health-check dependency set?
Include dependencies whose absence means the instance should not receive relevant traffic. Exclude noncritical dependencies or expose degraded status separately to avoid unnecessary fleet-wide unready states.

### Q37. How do you prevent probes from masking a broken service?
Pair health endpoints with real request metrics, synthetic checks, and dependency telemetry. A lightweight probe is not proof that every business path works.

### Q38. What does observability cardinality mean operationally?
Unique label combinations create time series and cost. Use bounded labels for metrics and place request-level identity in traces/logs.

### Q39. How should logs be structured?
Use consistent fields for severity, service, operation, trace ID, and safe business correlation. Redact sensitive values and avoid duplicate stack traces.

### Q40. How do you investigate an incident without overfitting to one signal?
Correlate independent evidence—metrics, traces, logs, thread dumps, and database telemetry. Confirm hypotheses with a targeted experiment and track whether mitigation improves user-visible outcomes.

### Q41. What is a good rollback criterion?
A measurable breach in error/latency/resource or correctness signals, combined with an understanding of schema compatibility and side effects. Decide thresholds before rollout.

### Q42. Why can rollback be unsafe after a database migration?
New schema or data transformations may be incompatible with old code. Use backward-compatible migrations or a forward-recovery plan.

### Q43. What is the value of a canary?
It limits exposure while comparing new-version behavior under real traffic. It requires representative routing, sufficient observation time, and version-correlated telemetry.

### Q44. How should dependency versions be managed?
Use a coherent BOM/release train, review overrides, scan transitive dependencies, and test the resolved graph. Avoid unreviewed version drift.

### Q45. How do you evaluate a framework upgrade?
Compare migration cost and support/security needs against risk, test the complete integration surface, and plan staged deployment. Do not upgrade only because a new major version exists.

### Q46. How do you distinguish configuration failure from code failure?
Inspect effective properties, profiles, condition reports, dependency graph, and the first causal exception. Reproduce with a minimal context before changing business logic.

### Q47. What is the danger of relying on framework defaults?
Defaults may change across versions and may not match business/security needs. Know which defaults are relied upon and make critical policies explicit.

### Q48. What is a useful design review question for every integration?
What happens when it is slow, unavailable, returns invalid data, or succeeds but the caller times out? This exposes timeout, retry, idempotency, and recovery requirements.

### Q49. How should senior engineers discuss uncertainty?
State what is known, what depends on version/configuration, what evidence is missing, and how to verify it. Do not convert a plausible assumption into a guarantee.

### Q50. What makes a Spring solution production-grade?
Correct behavior plus explicit failure semantics, bounded resource use, security, observability, repeatable tests, compatible upgrades, and operational recovery.
