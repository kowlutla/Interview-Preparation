# Senior Production Scenarios 56–65

## Scenario 56 — Duplicate bean after adding a starter

**Symptoms:** Context fails with duplicate or ambiguous bean errors.

**Investigate:** Check auto-configuration conditions, component scanning, explicit @Bean definitions, and library versions.

**Resolve:** Remove duplicate registration or use a deliberate conditional customization. Do not disable all auto-configuration to suppress one conflict.

**Prevent:** Test the starter with and without application overrides.

**Follow-up:** Why can adding a dependency change the context even if application code is unchanged?

## Scenario 57 — Application returns 500 for a client error

**Symptoms:** Invalid input becomes a generic server failure.

**Investigate:** Inspect binding/validation exception types, handler selection, and advice ordering.

**Resolve:** Map expected request errors to stable client responses and keep unexpected exceptions as 500.

**Prevent:** Contract-test malformed JSON, invalid fields, and business-rule failures separately.

**Follow-up:** Why is mapping every exception to 400 misleading?

## Scenario 58 — Slow serialization dominates latency

**Symptoms:** Database spans are fast but response latency remains high.

**Investigate:** Profile serialization, payload size, entity graph traversal, custom serializers, and repeated computed properties.

**Resolve:** Return purpose-built DTOs, reduce payload size, and optimize only measured hotspots.

**Prevent:** Track response size and test serialization of representative data.

**Follow-up:** How can returning entities trigger additional database queries?

## Scenario 59 — Database migration blocks deployment

**Symptoms:** Startup migration takes too long or locks production tables.

**Investigate:** Review migration statements, lock behavior, table size, transaction strategy, and whether every replica attempts startup migration.

**Resolve:** Use staged expand-and-contract changes, pre-deployment migration jobs where appropriate, and tested rollback/recovery plans.

**Prevent:** Exercise migrations against production-like data volumes.

**Follow-up:** Why can a schema change break rolling deployments?

## Scenario 60 — New deployment causes connection storm

**Symptoms:** Database connections spike as replicas start simultaneously.

**Investigate:** Check pool max size per replica, startup probes, retry behavior, connection initialization, and total replica count during rolling deployment.

**Resolve:** Budget aggregate connections, stagger rollout, and tune pool sizes against database limits.

**Prevent:** Load-test rollout surge capacity.

**Follow-up:** Why must pool size be considered across all replicas?

## Scenario 61 — A custom filter runs in the wrong order

**Symptoms:** Security or request context is missing when a filter executes.

**Investigate:** Inspect servlet filter registration, security filter chain ordering, and whether the request matches a different chain.

**Resolve:** Place the filter at the correct pipeline boundary and use supported ordering APIs.

**Prevent:** Integration-test the actual filter sequence and failure behavior.

**Follow-up:** How does a servlet filter differ from an MVC interceptor?

## Scenario 62 — Request context leaks between tasks

**Symptoms:** Wrong tenant or trace context appears in logs from pooled threads.

**Investigate:** Check ThreadLocal cleanup, executor reuse, and unsupported context propagation assumptions.

**Resolve:** Use framework-supported context propagation or explicit immutable task context; always clear manually managed thread-local state in finally blocks.

**Prevent:** Add concurrent tests with different tenants and tracing identities.

**Follow-up:** Why are thread pools a risk for ThreadLocal leaks?

## Scenario 63 — CORS works locally but fails in production

**Symptoms:** Browser preflight fails only through gateway or ingress.

**Investigate:** Check origin, scheme, headers, credentials, proxy behavior, and whether gateway and service apply conflicting policies.

**Resolve:** Establish one coherent policy at the intended boundary and ensure preflight requests are handled correctly.

**Prevent:** Test deployed-like hostnames and HTTPS configuration.

**Follow-up:** Does CORS prevent non-browser clients from calling the API?

## Scenario 64 — Rate limiter blocks legitimate traffic

**Symptoms:** Users receive 429 during bursts or multi-instance traffic.

**Investigate:** Review key selection, burst/window algorithm, shared state, clock assumptions, and client identity.

**Resolve:** Tune quota semantics to product requirements and return appropriate retry guidance where supported.

**Prevent:** Test burst, sustained rate, and multi-replica cases.

**Follow-up:** Why can an in-memory limiter behave inconsistently behind a load balancer?

## Scenario 65 — Async queue delay rises before errors

**Symptoms:** Background tasks are increasingly stale while the API still returns success.

**Investigate:** Monitor queue depth, oldest task age, worker utilization, rejection counts, and downstream latency.

**Resolve:** Apply backpressure or admission control, tune only after measuring capacity, and make task durability explicit.

**Prevent:** Alert on queue age, not just task failures.

**Follow-up:** Why can increasing queue size make user-visible latency worse?
