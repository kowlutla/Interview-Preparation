# Senior Production Scenarios 56–65

## Scenario 56 — Duplicate bean after adding a starter

**Symptoms:** Context fails with duplicate or ambiguous bean errors.

**Investigation:** Check auto-configuration conditions, component scanning, explicit @Bean definitions, and library versions.

**Resolution:** Remove duplicate registration or use a deliberate conditional customization. Do not disable all auto-configuration to suppress one conflict.

**Prevention:** Test the starter with and without application overrides.

**Follow-up:** Why can adding a dependency change the context even if application code is unchanged?

## Scenario 57 — Application returns 500 for a client error

**Symptoms:** Invalid input becomes a generic server failure.

**Investigation:** Inspect binding/validation exception types, handler selection, and advice ordering.

**Resolution:** Map expected request errors to stable client responses and keep unexpected exceptions as 500.

**Prevention:** Contract-test malformed JSON, invalid fields, and business-rule failures separately.

**Follow-up:** Why is mapping every exception to 400 misleading?

## Scenario 58 — Slow serialization dominates latency

**Symptoms:** Database spans are fast but response latency remains high.

**Investigation:** Profile serialization, payload size, entity graph traversal, custom serializers, and repeated computed properties.

**Resolution:** Return purpose-built DTOs, reduce payload size, and optimize only measured hotspots.

**Prevention:** Track response size and test serialization of representative data.

**Follow-up:** How can returning entities trigger additional database queries?

## Scenario 59 — Database migration blocks deployment

**Symptoms:** Startup migration takes too long or locks production tables.

**Investigation:** Review migration statements, lock behavior, table size, transaction strategy, and whether every replica attempts startup migration.

**Resolution:** Use staged expand-and-contract changes, pre-deployment migration jobs where appropriate, and tested rollback/recovery plans.

**Prevention:** Exercise migrations against production-like data volumes.

**Follow-up:** Why can a schema change break rolling deployments?

## Scenario 60 — New deployment causes connection storm

**Symptoms:** Database connections spike as replicas start simultaneously.

**Investigation:** Check pool max size per replica, startup probes, retry behavior, connection initialization, and total replica count during rolling deployment.

**Resolution:** Budget aggregate connections, stagger rollout, and tune pool sizes against database limits.

**Prevention:** Load-test rollout surge capacity.

**Follow-up:** Why must pool size be considered across all replicas?

## Scenario 61 — A custom filter runs in the wrong order

**Symptoms:** Security or request context is missing when a filter executes.

**Investigation:** Inspect servlet filter registration, security filter chain ordering, and whether the request matches a different chain.

**Resolution:** Place the filter at the correct pipeline boundary and use supported ordering APIs.

**Prevention:** Integration-test the actual filter sequence and failure behavior.

**Follow-up:** How does a servlet filter differ from an MVC interceptor?

## Scenario 62 — Request context leaks between tasks

**Symptoms:** Wrong tenant or trace context appears in logs from pooled threads.

**Investigation:** Check ThreadLocal cleanup, executor reuse, and unsupported context propagation assumptions.

**Resolution:** Use framework-supported context propagation or explicit immutable task context; always clear manually managed thread-local state in finally blocks.

**Prevention:** Add concurrent tests with different tenants and tracing identities.

**Follow-up:** Why are thread pools a risk for ThreadLocal leaks?

## Scenario 63 — CORS works locally but fails in production

**Symptoms:** Browser preflight fails only through gateway or ingress.

**Investigation:** Check origin, scheme, headers, credentials, proxy behavior, and whether gateway and service apply conflicting policies.

**Resolution:** Establish one coherent policy at the intended boundary and ensure preflight requests are handled correctly.

**Prevention:** Test deployed-like hostnames and HTTPS configuration.

**Follow-up:** Does CORS prevent non-browser clients from calling the API?

## Scenario 64 — Rate limiter blocks legitimate traffic

**Symptoms:** Users receive 429 during bursts or multi-instance traffic.

**Investigation:** Review key selection, burst/window algorithm, shared state, clock assumptions, and client identity.

**Resolution:** Tune quota semantics to product requirements and return appropriate retry guidance where supported.

**Prevention:** Test burst, sustained rate, and multi-replica cases.

**Follow-up:** Why can an in-memory limiter behave inconsistently behind a load balancer?

## Scenario 65 — Async queue delay rises before errors

**Symptoms:** Background tasks are increasingly stale while the API still returns success.

**Investigation:** Monitor queue depth, oldest task age, worker utilization, rejection counts, and downstream latency.

**Resolution:** Apply backpressure or admission control, tune only after measuring capacity, and make task durability explicit.

**Prevention:** Alert on queue age, not just task failures.

**Follow-up:** Why can increasing queue size make user-visible latency worse?
