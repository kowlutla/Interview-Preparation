# Mock Interview 3 — Senior Backend Engineer

## 1. Explain proxy-based transaction management.

Spring commonly wraps eligible beans in proxies that intercept external calls and apply transaction semantics. Self-invocation bypasses the proxy; proxy type, method visibility, exception propagation, and transaction manager selection all matter.

**Follow-up:** How would you prove the transaction boundary? Use transaction logs and an integration test that asserts rollback after a forced failure.

## 2. A service updates a database then publishes an event. What can fail?

The database commit and broker publish are separate operations, so a crash can leave state committed without an event or publish an event for a rolled-back change.

**Preferred design:** Transactional outbox plus a retrying relay and idempotent consumers.

**Follow-up:** What operational signal detects a stuck outbox? Oldest pending event age and relay failure metrics.

## 3. How do you diagnose connection pool exhaustion?

Correlate pool active/pending metrics with query latency, transaction duration, lock waits, request concurrency, and thread dumps. Fix resource holding and database bottlenecks before increasing pool capacity.

**Follow-up:** Why might a bigger pool make latency worse? It can increase database contention and overload.

## 4. How do you design JWT authorization?

Use resource-server support, validate issuer/signature/expiry and required audience claims, map authorities deliberately, and enforce tenant/resource authorization at service boundaries.

**Follow-up:** What is wrong with trusting a decoded JWT payload? Decoding alone does not verify the signature or claims.

## 5. How do you choose MVC versus WebFlux?

Choose based on workload and end-to-end I/O model. MVC is often simpler with blocking JPA; WebFlux can help high-concurrency non-blocking workloads but blocking calls on event loops defeat its design.

**Follow-up:** How would you verify the choice? Benchmark representative traffic and failure conditions.

## 6. How do you handle a slow downstream dependency?

Set end-to-end deadlines, bounded client timeouts, carefully scoped retries, circuit breakers/bulkheads where justified, and safe fallbacks for noncritical features.

**Follow-up:** What happens when several layers retry? Attempts multiply and can amplify overload.

## 7. How do you plan a Boot major upgrade?

Read migration notes, align Java and BOM versions, audit Jakarta namespace changes and removed APIs, then test startup, security, persistence, outbound clients, observability, and shutdown. Roll out incrementally.

**Follow-up:** Why check Spring Cloud compatibility separately? Cloud components follow release-train compatibility constraints.

## 8. How do you find a memory leak?

Compare post-GC heap trend, allocation profile, heap-dump dominators, queue/cache growth, thread counts, and native memory. Identify retained references before changing heap size.

**Follow-up:** Can native memory exhaust the container while heap remains below max? Yes.

## 9. How do you make an API multi-tenant safe?

Derive tenant identity from trusted authentication, enforce tenant predicates at service/data boundaries, scope cache keys by tenant, and test cross-tenant access.

**Follow-up:** Why is a client-supplied tenant header not sufficient? It can be forged unless bound to trusted identity and authorization.

## 10. What makes a good production incident response?

Establish impact and timeline, use metrics/traces/logs to localize the bottleneck, mitigate safely, preserve evidence, and follow up with tests and prevention. Avoid speculative changes without a rollback path.

**Follow-up:** How do you decide whether to roll back? Compare user impact, risk of continued operation, migration compatibility, and recovery options.
