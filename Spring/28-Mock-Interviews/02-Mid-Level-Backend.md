# Mock Interview 2 — Mid-Level Backend Engineer

## 1. How does a REST request reach a controller?

In Spring MVC, DispatcherServlet coordinates handler mapping, argument resolution, invocation, message conversion, and exception handling. Filters and security may process the request before controller invocation.

**Follow-up:** Where should CORS be configured? At the appropriate web/security boundary based on the application setup.

## 2. How do you validate request input?

Use Jakarta Bean Validation constraints on DTOs and trigger validation at the controller boundary. Keep business invariants in the application/domain layer as well.

**Follow-up:** Should a database uniqueness constraint be replaced by request validation? No; concurrent requests can race, so the database constraint remains authoritative.

## 3. How do you implement consistent API errors?

Use a stable error contract and centralized advice for known exceptions. Return semantically appropriate statuses and keep internal diagnostics out of client responses.

**Follow-up:** How would you correlate the error with logs? Include a trace/correlation identifier.

## 4. What does @Transactional do?

It declares transaction semantics applied by transaction infrastructure, commonly through a proxy. Rollback depends on exception rules and transaction manager behavior.

**Follow-up:** Why might it fail on self-invocation? The internal call bypasses the proxy.

## 5. How would you implement pagination?

Use bounded page size and deterministic sorting. Consider cursor/keyset pagination for deep pages and high-volume changing datasets.

**Follow-up:** Why is sorting by a non-unique timestamp risky? Equal timestamps can lead to unstable page boundaries.

## 6. How do you test a controller?

Use MockMvc with a focused MVC slice to verify routing, binding, validation, status codes, and JSON contract. Add full integration tests for wiring and persistence behavior.

**Follow-up:** What does the slice test not prove? That the entire production application context starts correctly.

## 7. How do you call another service reliably?

Configure timeouts, connection pooling, error mapping, and retries only for safe transient failures. Instrument latency and outcomes.

**Follow-up:** Why can retrying a POST be dangerous? The remote operation may have succeeded despite a timeout.

## 8. How do you cache a read?

Use a stable key and clear invalidation semantics, and understand proxy interception. Select local versus distributed cache based on sharing and consistency needs.

**Follow-up:** What is a cache stampede? Concurrent misses trigger many simultaneous recomputations.

## 9. What would you monitor in production?

Request rate, error rate, latency percentiles, resource use, database pool wait, downstream latency, and readiness. Use logs and traces to diagnose individual failures.

**Follow-up:** Why avoid user IDs in metric labels? They create high cardinality.

## 10. How do you prevent duplicate scheduled work?

Use distributed coordination if only one replica should run it, and make the work idempotent. Local synchronization does not coordinate separate processes.

**Follow-up:** Is a distributed lock enough for exactly-once effects? No; crashes and retries still require idempotency.
