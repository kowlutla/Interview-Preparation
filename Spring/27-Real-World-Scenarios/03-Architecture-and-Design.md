# Production Scenarios — Architecture and Design

## Scenario 21 — Design a multi-tenant API

**Requirements:** Every request must be isolated to the authenticated tenant.

**Reasoning:** Derive tenant identity from a trusted authenticated claim or server-side mapping, not an arbitrary client header alone. Enforce tenant scoping in service/data access boundaries, include tenant-aware cache keys, and apply authorization to cross-tenant administrative paths explicitly.

**Failure modes:** Missing tenant predicates, cache key collisions, background jobs without tenant context, and privileged endpoints that bypass normal checks.

**Validation:** Test cross-tenant reads and writes, forged tenant headers, async jobs, and cache isolation.

**Follow-up:** Where should tenant context live, and how is it propagated safely?

## Scenario 22 — Design a consistent API error contract

**Requirements:** Several teams own controllers but clients need stable errors.

**Reasoning:** Define a shared schema and error-code taxonomy, use central exception mapping for framework and application errors, and keep domain exceptions independent of HTTP. Include correlation metadata but never expose stack traces.

**Alternatives:** A shared library can standardize types, while a platform gateway can normalize selected edge errors. Avoid hiding service-specific semantics behind an overly generic code.

**Validation:** Contract tests across representative endpoints and negative-path cases.

**Follow-up:** How do you evolve the error schema without breaking clients?

## Scenario 23 — Choose MVC or WebFlux

**Requirements:** A service mostly calls blocking JPA repositories and a few remote services.

**Reasoning:** Spring MVC is often simpler for blocking persistence workloads. WebFlux is justified when the end-to-end path benefits from non-blocking I/O and concurrency profile. A reactive controller calling blocking JPA does not become non-blocking by declaration.

**Trade-offs:** Team expertise, library support, context propagation, debugging, and load characteristics matter.

**Validation:** Benchmark realistic traffic and failure scenarios, including tail latency and resource consumption.

**Follow-up:** Under what workload would WebFlux become compelling?

## Scenario 24 — Design multiple data sources

**Requirements:** Operational writes go to one database and analytics reads to another.

**Reasoning:** Define separately qualified DataSource and transaction manager configurations, make repository/entity-manager ownership explicit, and avoid implying a local transaction spans both stores.

**Failure modes:** Wrong transaction manager selected, repositories wired to the wrong source, inconsistent cross-database writes, and migration mismatch.

**Validation:** Context wiring tests and integration tests for each resource.

**Follow-up:** How would you coordinate a business operation that must update both databases?

## Scenario 25 — Design reliable event publishing

**Requirements:** Every committed order must eventually produce an integration event.

**Reasoning:** Write order and outbox record in one transaction. A relay publishes with retries and tracks progress. Consumers deduplicate by stable event identity and handle reordering according to domain rules.

**Trade-offs:** Eventual publication, relay operations, and duplicate handling are explicit costs.

**Validation:** Crash at each boundary, retry publication, and replay duplicate messages.

**Follow-up:** How would you detect a stuck outbox relay?

## Scenario 26 — Design service-to-service resilience

**Requirements:** An API calls three downstream services with different criticality.

**Reasoning:** Establish an overall deadline, allocate per-call budgets, limit concurrency, and apply retry/circuit-breaker behavior based on idempotency and dependency criticality. Use fallbacks only where business semantics allow them.

**Failure modes:** Retry storms, thread exhaustion, partial results mistaken for complete results, and cascading timeouts.

**Validation:** Inject latency, errors, and partial outages under load.

**Follow-up:** How do you keep retries from violating the caller's deadline?

## Scenario 27 — Design observability for an API

**Requirements:** On-call engineers must identify high latency and dependency failures.

**Reasoning:** Use RED metrics (rate, errors, duration), traces across downstream calls, structured logs with trace IDs, and dashboards tied to user-visible SLOs. Keep metric labels low-cardinality and protect sensitive fields.

**Trade-offs:** Trace sampling and telemetry volume need budgets; metrics alone do not explain every request.

**Validation:** Simulate a database bottleneck and verify the signals identify the layer.

**Follow-up:** Why should raw user IDs not be metric labels?

## Scenario 28 — Design safe production configuration

**Requirements:** Different environments need distinct endpoints, limits, and credentials.

**Reasoning:** Externalize settings, validate them at startup, store secrets in an approved secret manager, and document precedence. Use a controlled rollout for configuration changes; do not assume arbitrary beans refresh automatically.

**Failure modes:** Secret leakage, wrong profile, silent property override, and invalid pool limits.

**Validation:** Test missing/invalid properties and review deployment manifests for accidental secrets.

**Follow-up:** How would you determine which property source won?

## Scenario 29 — Design graceful shutdown for a consumer service

**Requirements:** Deployments should not lose in-flight work.

**Reasoning:** Stop accepting new work, coordinate consumer pause/acknowledgment, finish bounded in-flight tasks, and persist recoverable state. Assume redelivery is possible and make processing idempotent.

**Trade-offs:** Longer shutdown reduces abrupt interruption but affects rollout time; bounded deadlines require a recovery strategy for unfinished work.

**Validation:** Terminate under load and verify redelivery without duplicate business effects.

**Follow-up:** Why is graceful shutdown not a guarantee of exactly-once processing?

## Scenario 30 — Upgrade a mature Boot application

**Requirements:** Upgrade across a major version with low operational risk.

**Reasoning:** Inventory versions, read migration notes, align Java and BOMs, remove deprecated APIs, migrate namespaces where needed, and build smoke coverage for security, persistence, clients, metrics, and shutdown. Roll out gradually with rollback.

**Failure modes:** Hidden transitive incompatibility, changed property semantics, security rule differences, and missing Jakarta-compatible libraries.

**Validation:** Test on production-like infrastructure and compare key operational metrics before and after.

**Follow-up:** Which checks belong in pre-production smoke tests versus canary monitoring?
