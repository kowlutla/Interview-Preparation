# Mock Interview 5 — Senior Product-Based Company

## 1. A request occasionally commits twice. How do you investigate?

Build a timeline across client retries, load balancer retries, application logs, transaction commits, and downstream calls. Determine whether duplicate requests share an idempotency key and whether the database enforces uniqueness.

**Solution direction:** Durable idempotency at the business boundary, not only a frontend button disable.

## 2. The API is fast at p50 but slow at p99. What do you inspect?

Look for queueing, pool acquisition delays, long-tail database queries, downstream timeouts, GC pauses, lock contention, and uneven load. Use traces and percentile-aware metrics rather than averages alone.

## 3. How would you design a resilient checkout flow?

Separate critical payment/inventory invariants from optional features, use bounded synchronous calls, durable state transitions, idempotency, and an explicit compensation/reconciliation strategy for distributed outcomes.

**Follow-up:** What if payment times out after the provider charges the customer? Reconcile using provider transaction/idempotency identity; do not assume failure.

## 4. How do you keep a Spring singleton safe under concurrency?

Avoid mutable request-specific fields. Use immutable dependencies, thread-safe shared components, and explicit synchronization/atomic structures only where shared mutation is required.

## 5. Why might a @Transactional method not be transactional?

The invocation may bypass the proxy, the bean may not be managed, the method may not be intercepted under the chosen proxy model, transaction management may be absent, or a different transaction manager may be active.

## 6. How do you secure a multi-tenant data API?

Authenticate through trusted identity, derive tenant context from validated claims, enforce tenant authorization in service and query boundaries, and ensure background tasks and caches preserve isolation.

## 7. How do you upgrade Spring with minimal risk?

Inventory dependencies and APIs, read official migration docs, stage the upgrade, run focused and production-like tests, deploy canary replicas, monitor key SLOs, and keep a tested rollback strategy compatible with schema changes.

## 8. When is WebFlux the wrong choice?

When most operations rely on blocking libraries and concurrency needs do not justify the reactive complexity. Mixing blocking persistence with event-loop processing can increase operational difficulty without benefits.

## 9. What is your response to an outage caused by connection pool exhaustion?

Mitigate request load and unsafe retries, inspect query/transaction and pool metrics, identify held connections and database limits, then tune only after removing root causes. Communicate impact and verify recovery through metrics.

## 10. How do you distinguish a framework bug from application misuse?

Create a minimal reproduction, identify exact versions, inspect stack traces and source/docs, verify proxy and configuration assumptions, and compare behavior against supported contracts. Avoid declaring a framework defect from symptoms alone.

## 11. How do you decide whether to cache?

Define freshness and consistency requirements, identify the expensive source, estimate hit ratio and invalidation complexity, and measure end-to-end impact. A cache adds operational and correctness costs.

## 12. What distinguishes a senior answer?

A senior answer states assumptions, explores failure modes, gives a measurable validation plan, considers security and operations, and compares alternatives rather than presenting one annotation as a universal fix.
