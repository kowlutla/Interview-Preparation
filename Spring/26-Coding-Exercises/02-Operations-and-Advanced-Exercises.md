# Coding Exercises — Advanced and Operations

These exercises complement [Core and REST exercises](01-Core-and-REST-Exercises.md). They focus on production behavior and require explicit assumptions, tests, and trade-offs.

## Exercise 26 — Consistent Problem Details

**Problem:** Standardize API errors across controllers.

**Requirements:** Stable type/title/status fields, field-level validation details, trace correlation, and no internal stack traces.

**Reference approach:** Centralize known exception mapping in `@RestControllerAdvice`; add a response filter or request context integration for correlation IDs. Do not expose arbitrary exception messages.

**Validation:** Test business, validation, unknown, and downstream exceptions.

## Exercise 27 — Multi-data-source routing

**Problem:** Connect a service to a primary and reporting database.

**Reference approach:** Define separately named DataSource and transaction manager beans, qualify repository/entity-manager configuration, and decide which transaction manager owns each operation.

**Common mistake:** Assuming one `@Transactional` spans two independent databases.

**Validation:** Test reads/writes against both stores and failure between separate transaction boundaries.

## Exercise 28 — Transaction synchronization

**Problem:** Trigger an in-process action only after a successful transaction commit.

**Reference approach:** Use `@TransactionalEventListener(phase = AFTER_COMMIT)` when the action is an in-process reaction. For durable cross-process delivery, use an outbox instead.

**Validation:** Verify no listener action on rollback and expected behavior when no transaction is active.

## Exercise 29 — Idempotent message consumer

**Problem:** Process a message at least once without applying its business effect twice.

**Reference approach:** Persist a unique event identifier and business change atomically, with a uniqueness constraint and explicit duplicate handling.

**Common mistake:** Keeping deduplication IDs only in memory.

**Validation:** Deliver the same message repeatedly and concurrently; verify one business effect.

## Exercise 30 — Retry with backoff

**Problem:** Retry a transient downstream call within a strict deadline.

**Reference approach:** Classify retryable failures, cap attempts, add backoff/jitter, and ensure the combined attempt durations fit the caller's deadline. Ensure operation idempotency.

**Validation:** Test transient recovery, permanent failure, and deadline exhaustion.

## Exercise 31 — Circuit-breaker fallback

**Problem:** Protect a service from a failing dependency.

**Reference approach:** Configure a supported circuit-breaker integration with measured thresholds, bounded timeouts, and a fallback that is semantically safe. Do not return fabricated success for critical writes.

**Validation:** Simulate open, half-open, and closed states and observe metrics.

## Exercise 32 — Rate-limited API

**Problem:** Enforce a per-client request limit across multiple replicas.

**Reference approach:** Use a shared limiter or gateway policy with explicit identity, burst, and failure behavior. Avoid a process-local counter when the limit must be global.

**Validation:** Test concurrency across instances and limiter-storage outage behavior.

## Exercise 33 — Structured logging

**Problem:** Emit structured logs with trace correlation.

**Reference approach:** Configure structured logging for the selected Boot version, propagate supported trace context, and redact sensitive fields at source. Avoid duplicate logging of the same exception at every layer.

**Validation:** Verify JSON parseability, correlation across calls, and secret redaction.

## Exercise 34 — Graceful executor shutdown

**Problem:** Stop accepting async tasks during shutdown and allow bounded completion.

**Reference approach:** Configure executor lifecycle and await-termination behavior within the platform termination grace period. Define whether remaining work is persisted, retried, or discarded.

**Validation:** Terminate under queued workload and verify the policy.

## Exercise 35 — Upload limits

**Problem:** Safely accept file uploads.

**Reference approach:** Enforce size limits at ingress and application layers, validate authorization/content, generate server-side filenames, and stream large content to controlled storage.

**Validation:** Test oversize, empty, malformed, and unauthorized uploads.

## Exercise 36 — Request correlation

**Problem:** Return a correlation ID and include it in logs.

**Reference approach:** Prefer established tracing instrumentation where possible. If accepting a caller-provided ID, validate its length and characters, prevent log injection, and establish trust rules.

**Validation:** Verify IDs propagate through synchronous calls and are not used as high-cardinality metric labels.

## Exercise 37 — Safe configuration exposure

**Problem:** Provide diagnostics without leaking credentials.

**Reference approach:** Restrict Actuator exposure, secure sensitive endpoints, and redact secrets in logs and support bundles.

**Validation:** Assert unauthorized access is denied and known secret values never appear in diagnostic output.

## Exercise 38 — Readiness policy

**Problem:** Keep an instance out of traffic when a critical dependency is unavailable, without causing restart loops.

**Reference approach:** Model dependency health in readiness and keep liveness focused on process progress. Bound checks and define dependency criticality explicitly.

**Validation:** Simulate dependency outage and verify readiness changes while liveness remains appropriate.

## Exercise 39 — Startup runner

**Problem:** Load a small required reference dataset during startup.

**Reference approach:** Use an application runner only when the startup work is bounded and readiness should wait for it. For large imports, use a separately controlled job with progress and recovery.

**Validation:** Test failure behavior, duplicate execution, and startup readiness.

## Exercise 40 — Repository query count

**Problem:** Detect an N+1 regression in a service.

**Reference approach:** Add an integration test with representative entities, measure SQL/query count using test instrumentation, and fix with projection, entity graph, or query redesign.

**Validation:** Ensure the test fails when per-row lazy queries return.

## Exercise 41 — Optimistic locking conflict

**Problem:** Handle concurrent updates to the same record.

**Reference approach:** Use a version column and translate a conflict into a documented retry or conflict response. Never blindly retry a business command without re-evaluating the latest state.

**Validation:** Run two concurrent transactions and assert one detects the conflict.

## Exercise 42 — Test async behavior

**Problem:** Test a method returning CompletableFuture.

**Reference approach:** Await the future with a bounded timeout, assert success/failure explicitly, and use a controllable executor or fake dependency to make timing deterministic.

**Common mistake:** Adding arbitrary Thread.sleep calls.

**Validation:** Repeat the test under CI load and ensure failures have useful diagnostics.

## Exercise 43 — Test secured method authorization

**Problem:** Verify a service operation requires a specific authority.

**Reference approach:** Enable method security in the test context, invoke through the Spring proxy, and test allowed and denied principals.

**Common mistake:** Directly constructing the service, thereby bypassing method-security interception.

**Validation:** Confirm a denied invocation never reaches the side effect.

## Exercise 44 — Configuration precedence diagnosis

**Problem:** A production property differs from the value in application.yaml.

**Reference approach:** Inspect active profiles and property sources, then identify environment variables, command-line arguments, deployment injection, and imported config that override the file.

**Validation:** Add a configuration binding test and document the intended source of truth.

## Exercise 45 — Boot auto-configuration test

**Problem:** Test a starter's default bean and user override.

**Reference approach:** Use Boot's auto-configuration test support with isolated contexts, include the configuration under test, and test both the conditional default and a user-defined replacement.

**Validation:** Assert bean types/counts and test missing-class/property conditions.

## Exercise 46 — Outbound authentication

**Problem:** Add service authentication to a REST client.

**Reference approach:** Use a client interceptor/filter or supported OAuth client integration with scoped credentials and token lifecycle management. Do not hardcode secrets or forward inbound tokens without audience/trust validation.

**Validation:** Test missing credentials, expired token refresh, and destination allow-listing.

## Exercise 47 — Reactive context

**Problem:** Preserve trace metadata across a reactive pipeline.

**Reference approach:** Use supported tracing instrumentation and Reactor Context integration. Avoid assuming ThreadLocal state survives scheduler hops.

**Validation:** Assert trace context is available after a thread switch and no user-specific data leaks between subscriptions.

## Exercise 48 — Cache stampede mitigation

**Problem:** Avoid hundreds of concurrent recomputations for one expired key.

**Reference approach:** Evaluate cache-provider-supported synchronization, request coalescing, staggered expiry, or controlled refresh. Keep the source call bounded and define behavior during cache failure.

**Validation:** Launch concurrent requests and measure source-call count and tail latency.

## Exercise 49 — Memory-growth investigation

**Problem:** A long-running service's memory rises over several hours.

**Reference approach:** Compare post-GC heap, allocation profile, heap dump dominators, queue/cache sizes, thread count, and native memory. Identify retained objects before changing heap size.

**Validation:** Reproduce under load and verify retained memory stabilizes after the fix.

## Exercise 50 — Upgrade compatibility smoke suite

**Problem:** Upgrade Boot across a major version.

**Reference approach:** Read the migration guide, align Java and managed dependencies, resolve namespace/API changes, and create smoke tests for startup, security, persistence, clients, metrics, and graceful shutdown.

**Validation:** Run tests against a production-like database and broker; capture startup and condition reports for failures.
