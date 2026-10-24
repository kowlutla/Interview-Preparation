# Mock Interview 4 — Microservices Engineer

## 1. What is Spring Cloud?

It provides integrations for distributed-system concerns such as gateways, discovery, centralized configuration, client-side load balancing, and resilience. Compatibility with the chosen Boot release must be checked.

## 2. What does an API gateway solve?

It centralizes edge routing and selected cross-cutting policies. It should not replace service-level authorization, and internal services should not be accidentally reachable through bypass paths.

## 3. How do timeouts and retries interact?

Timeouts bound waits; retries attempt selected transient failures. Use a total deadline, bounded attempts, backoff/jitter, and idempotency. Nested retry policies can multiply load.

## 4. What is a circuit breaker?

It temporarily stops calls to a failing dependency based on configured failure/slow-call observations, then probes recovery. It limits repeated calls but does not repair the dependency.

## 5. Explain the outbox pattern.

Persist business state and an event record in the same database transaction; a relay publishes the record later. Consumers must handle duplicates, and operators need visibility into relay lag and failed records.

## 6. How do you prevent duplicate consumer effects?

Use stable event identity and durable idempotency, ideally atomically with the business update. Broker-level delivery semantics alone do not guarantee exactly-once external side effects.

## 7. How should distributed tracing work?

Propagate trace context across HTTP and messaging with supported instrumentation. Use metrics for aggregate patterns, logs for contextual events, and traces for cross-service paths.

## 8. What happens if the service registry is stale?

Calls may target dead instances or fail to find new ones. Health propagation, client refresh behavior, DNS/cache timing, and timeout policy determine recovery. Platform-native service discovery may be sufficient in some deployments.

## 9. How do you handle configuration changes?

Externalize settings and understand whether the running bean observes changes. Most bound singleton configuration is not dynamically refreshed without explicit support; a controlled rollout is often safer.

## 10. How do you troubleshoot a cascading outage?

Map dependency graph and call deadlines, inspect thread/connection pool saturation, retry rates, and circuit states. Reduce amplification, shed noncritical load, and restore the dependency while preserving evidence.

**Follow-up:** Which mitigation would you apply first if retries are multiplying traffic? Disable or reduce unsafe retries and enforce bounded timeouts/concurrency.
