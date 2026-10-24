# Production Scenarios — Container, Web, and Transactions

For every incident, collect evidence before applying a fix. Avoid treating an annotation change as a diagnosis.

## Scenario 1 — Missing bean after package refactor

**Symptoms:** Application fails during context refresh with a missing-bean error.

**Investigation:** Check the main application's component-scan root, runtime classpath, active profiles, conditional configuration, and test-specific context. Inspect the first meaningful cause and Boot condition report.

**Resolution:** Place components under the intended root package or define explicit scanning/import configuration. Fix missing dependency or condition causes instead of registering duplicate fallback beans.

**Trade-offs/prevention:** Broad scans can increase startup and accidentally discover unrelated components. Add a context-start test and keep package boundaries deliberate.

**Follow-up:** Why might the bean exist in production but not in a slice test?

## Scenario 2 — Two beans match one injection point

**Symptoms:** NoUniqueBeanDefinitionException.

**Investigation:** Enumerate candidates and determine whether the consumer needs one specific implementation, a default, or all implementations.

**Resolution:** Use a semantic qualifier, a justified primary default, or collection injection.

**Trade-offs/prevention:** Bean-name-based accidental selection is fragile. Add tests for the intended candidate and document extension points.

**Follow-up:** When is injecting List<Strategy> better than @Primary?

## Scenario 3 — Circular dependency after refactor

**Symptoms:** Context creation fails or bean creation reports a circular reference.

**Investigation:** Draw the dependency graph and identify shared responsibilities or callbacks that make two services depend on each other.

**Resolution:** Extract a third collaborator, invert a dependency through a stable interface, or publish an application event when genuinely decoupled notification is appropriate.

**Trade-offs/prevention:** Lazy injection can mask the cycle while leaving design and runtime-order risks. Add architecture tests or module boundaries.

**Follow-up:** Why are constructor cycles particularly problematic?

## Scenario 4 — Transaction does not roll back

**Symptoms:** A database change persists despite an error.

**Investigation:** Check whether the call passed through the Spring proxy, whether the exception escaped, rollback rules, transaction manager, propagation, and whether the observed write used the same resource transaction.

**Resolution:** Correct the service boundary and rollback policy. Avoid catching and suppressing exceptions that must abort the operation.

**Trade-offs/prevention:** Broad rollback rules may be right for a domain operation but can roll back on exceptions intended for recovery. Test failure after each meaningful write.

**Follow-up:** Why can a checked exception behave differently from a RuntimeException?

## Scenario 5 — Self-invocation bypasses transaction advice

**Symptoms:** An annotated internal method executes without the expected transaction.

**Investigation:** Trace the Java call path and determine whether invocation enters through the proxy or calls the same instance directly.

**Resolution:** Move the operation to a separate bean or redesign the transactional boundary. Avoid relying on fragile self-proxy injection.

**Trade-offs/prevention:** Splitting services solely for annotations can create poor abstractions; split around a meaningful operation boundary.

**Follow-up:** Which other Spring features commonly use proxies?

## Scenario 6 — Validation response is inconsistent

**Symptoms:** Similar invalid requests return different JSON structures or status codes.

**Investigation:** Identify local handlers, global advice ordering, framework-generated errors, binding exceptions, and validation exception types for the selected version.

**Resolution:** Define a stable error schema and centralize known exception mappings. Preserve field-level details while redacting sensitive values.

**Trade-offs/prevention:** Do not flatten every exception to 400. Test malformed JSON, missing fields, business validation, and unexpected failures.

**Follow-up:** When would ProblemDetail be appropriate?

## Scenario 7 — Endpoint returns 401 instead of 403

**Symptoms:** Authenticated user receives 401 for a forbidden operation.

**Investigation:** Verify whether authentication was actually established, whether the token was valid, which filter chain matched, and how authorities were mapped. Check entry-point and access-denied handler configuration.

**Resolution:** Correct authentication and authorization rules separately; map failures according to the configured security contract.

**Trade-offs/prevention:** Custom handlers can obscure semantics if they return the same status for all failures. Add anonymous, authenticated-denied, and authorized tests.

**Follow-up:** Explain the difference between AuthenticationEntryPoint and AccessDeniedHandler.

## Scenario 8 — JWT accepted with wrong claims

**Symptoms:** A token from an unintended issuer or audience is accepted, or a valid token is rejected.

**Investigation:** Inspect issuer/JWK configuration, signature algorithm trust, expiry/not-before validation, audience requirements, clock skew, and authority conversion.

**Resolution:** Use Spring Security resource-server support and explicit validators for the trust contract. Never trust decoded claims before cryptographic validation.

**Trade-offs/prevention:** Key rotation and identity-provider outages need operational handling. Test invalid issuer, expiry, audience, signature, and authorities.

**Follow-up:** Why is decoding a JWT not equivalent to validating it?

## Scenario 9 — Connection pool exhausted

**Symptoms:** Requests wait for connections, latency spikes, and pool pending count rises.

**Investigation:** Correlate active/idle/pending metrics with transaction duration, slow queries, lock waits, thread dumps, and downstream calls executed while a connection is held.

**Resolution:** Fix long-held connections and slow queries, bound transaction scope, then tune pool size against database capacity.

**Trade-offs/prevention:** Increasing the pool can worsen database contention. Load-test realistic concurrency and alert on acquisition latency.

**Follow-up:** Why can a remote HTTP call inside a transaction be dangerous?

## Scenario 10 — REST API slow under load

**Symptoms:** Tail latency grows although average latency seems acceptable.

**Investigation:** Break down time into queueing, database, downstream calls, serialization, GC, and thread-pool wait. Inspect traces, percentiles, thread dumps, and pool metrics.

**Resolution:** Address the measured bottleneck, apply bounded timeouts, improve query/serialization paths, and control concurrency.

**Trade-offs/prevention:** Caching or adding threads may shift rather than solve the bottleneck. Compare throughput, p95/p99, errors, and resource use after changes.

**Follow-up:** What evidence distinguishes CPU saturation from I/O waiting?
