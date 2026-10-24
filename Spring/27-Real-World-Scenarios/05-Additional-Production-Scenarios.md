# Additional Production Scenarios — Continued

## Scenario 46 — Async security context missing

**Symptoms:** An async operation behaves as anonymous although the initiating request was authenticated.

**Investigation:** Check executor configuration, context propagation support, and whether the async boundary crosses threads.

**Resolution:** Use the supported security-context propagation mechanism where appropriate, or pass explicit immutable identity/context data. Do not propagate privileged state indiscriminately.

**Prevention:** Test authorized and unauthorized async execution.

**Follow-up:** Why should a worker not blindly trust caller-provided tenant metadata?

## Scenario 47 — Retry causes duplicate payment

**Symptoms:** A timeout is followed by a second charge even though the first may have succeeded.

**Investigation:** Determine whether timeout occurred before or after the provider committed the operation and whether idempotency keys are supported.

**Resolution:** Use provider-supported idempotency keys and reconcile uncertain outcomes instead of blindly retrying.

**Prevention:** Treat timeouts as unknown outcomes for non-idempotent operations.

**Follow-up:** Why does a timeout not prove the remote operation failed?

## Scenario 48 — Circuit breaker opens too often

**Symptoms:** Calls fail fast during minor transient errors.

**Investigation:** Review failure-rate window, slow-call threshold, minimum sample size, timeout settings, and traffic distribution.

**Resolution:** Tune from observed behavior and distinguish expected business errors from dependency failures.

**Prevention:** Dashboard circuit state and test low-volume behavior.

**Follow-up:** What is the risk of a circuit breaker with too little sample data?

## Scenario 49 — Gateway and service disagree on auth

**Symptoms:** Gateway accepts a request but the service denies it, or a bypass route exposes the service.

**Investigation:** Review trust boundaries, token validation at each hop, network reachability, route matchers, and forwarded identity headers.

**Resolution:** Keep service-side authorization for resource access, validate trust across hops, and restrict direct service exposure.

**Prevention:** Test gateway and direct-service paths.

**Follow-up:** Why is a gateway not a replacement for service authorization?

## Scenario 50 — Traces have broken parent-child links

**Symptoms:** Requests appear as unrelated traces across services.

**Investigation:** Check instrumentation versions, propagation headers, async context, messaging headers, and manual executor boundaries.

**Resolution:** Use compatible tracing instrumentation and context propagation; avoid generating new roots unnecessarily.

**Prevention:** Run an end-to-end tracing smoke test after upgrades.

**Follow-up:** Why should trace IDs not be metric labels?

## Scenario 51 — Metrics cardinality explodes

**Symptoms:** Monitoring costs rise and metrics become slow to query.

**Investigation:** Find labels containing user IDs, order IDs, full URLs, exception messages, or arbitrary request data.

**Resolution:** Replace with bounded categories and put unique identifiers in logs/traces.

**Prevention:** Review tag design and cardinality budgets.

**Follow-up:** Which signals are appropriate for high-cardinality identifiers?

## Scenario 52 — Health endpoint causes load spike

**Symptoms:** Orchestrator probes generate repeated expensive dependency queries.

**Investigation:** Inspect probe interval, number of replicas, health implementation, timeouts, and repeated checks.

**Resolution:** Keep checks lightweight, cache carefully where appropriate, and separate liveness from readiness. Avoid health checks that perform full business operations.

**Prevention:** Load-test probe traffic during a dependency incident.

**Follow-up:** Why should health checks be bounded?

## Scenario 53 — Graceful shutdown exceeds termination grace period

**Symptoms:** Pods are killed before in-flight work completes.

**Investigation:** Compare platform termination grace period, server drain behavior, executor shutdown, consumer stop timeout, and longest task duration.

**Resolution:** Align shutdown budgets, stop new work early, and make unfinished work recoverable.

**Prevention:** Test termination during long requests and active message processing.

**Follow-up:** What happens if shutdown waits forever?

## Scenario 54 — Startup runner repeats a non-idempotent action

**Symptoms:** Each deployment duplicates seed data or external side effects.

**Investigation:** Check runner execution on every replica and every restart.

**Resolution:** Make initialization idempotent, use migrations/unique constraints for static data, and move recurring work to a controlled job system.

**Prevention:** Test repeated startup and multi-replica deployment.

**Follow-up:** Why is CommandLineRunner not a one-time deployment hook?

## Scenario 55 — Cache outage overloads database

**Symptoms:** Database traffic spikes when Redis becomes unavailable.

**Investigation:** Check cache failure mode, request concurrency, fallback behavior, and cache hit ratio.

**Resolution:** Apply bounded fallback concurrency, timeouts, local fallback where appropriate, or fail safely for noncritical reads. Avoid unbounded fan-out to the source.

**Prevention:** Chaos-test cache outages and define a degradation policy.

**Follow-up:** When is fail-closed behavior safer than fail-open?
