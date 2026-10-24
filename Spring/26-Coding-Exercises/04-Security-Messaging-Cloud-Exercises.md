# Coding Exercises — Security, Messaging, and Cloud

## Exercise 71 — Role-to-authority mapping

**Problem:** A token contains scopes but the API expects specific authorities.

**Reference approach:** Configure a deliberate JWT authority converter that maps the issuer's documented claims to Spring Security authorities. Keep the mapping explicit and test it against real claim shapes.

**Common mistake:** Assuming every identity provider uses the same roles claim.

**Validation:** Test missing scopes, multiple scopes, wrong claim type, and allowed/denied endpoint access.

## Exercise 72 — CSRF decision record

**Problem:** Decide whether CSRF protection is needed for a browser-facing API.

**Reference approach:** Document how credentials are stored and sent, whether the browser attaches them automatically, and which endpoints use cookie/session authentication versus bearer tokens. Configure CSRF based on the threat model, not because the service is labelled REST.

**Validation:** Test unsafe cross-site requests for cookie-authenticated flows.

## Exercise 73 — CORS allow-list

**Problem:** Permit a known frontend origin without allowing arbitrary origins.

**Reference approach:** Configure an explicit origin/method/header policy at the appropriate MVC/security boundary. Treat credentials as a separate decision and avoid wildcard origins with credentials.

**Validation:** Test preflight and denied-origin responses; remember CORS is not server-side authorization.

## Exercise 74 — Consumer retry policy

**Problem:** Retry transient message failures and route poison messages to a dead-letter destination.

**Reference approach:** Configure the Spring Kafka error-handling mechanism supported by the selected version, classify failures, bound attempts, and preserve event identity.

**Validation:** Test transient recovery, non-retryable failure, dead-letter routing, and replay.

## Exercise 75 — Kafka consumer idempotency

**Problem:** Avoid duplicate business effects when a message is redelivered.

**Reference approach:** Use a durable event ID with a database uniqueness constraint or an idempotent domain operation. Couple deduplication and state changes transactionally where possible.

**Validation:** Process duplicates sequentially and concurrently, including a simulated crash after the side effect.

## Exercise 76 — Outbox relay

**Problem:** Publish outbox records without losing events after a process crash.

**Reference approach:** Poll or stream pending records, publish with a stable event key, and track progress with retryable states. Design for duplicate publication and add metrics for oldest pending event age.

**Validation:** Crash between publish and mark-sent, restart, and verify consumer idempotency.

## Exercise 77 — Gateway route

**Problem:** Route an external path to an internal service while preserving authentication policy.

**Reference approach:** Define explicit route predicates and filters in the selected Spring Cloud Gateway version. Ensure authorization, header sanitization, timeout policy, and route ordering are intentional.

**Validation:** Test matching/nonmatching paths, authentication failure, downstream timeout, and sensitive-header removal.

## Exercise 78 — Client-side load balancing

**Problem:** Call a logical service name and distribute requests among instances.

**Reference approach:** Use the supported Spring Cloud load-balancer integration and a compatible discovery or instance source. Keep per-instance health and timeout behavior visible.

**Validation:** Test instance removal, stale discovery, and downstream failure.

## Exercise 79 — Circuit-breaker metrics

**Problem:** Measure dependency failure and circuit state.

**Reference approach:** Integrate a supported circuit-breaker library, expose low-cardinality metrics, and choose thresholds from observed latency/error patterns rather than arbitrary defaults.

**Validation:** Force state transitions and verify alerts do not fire from a single harmless transient failure.

## Exercise 80 — Scheduled-job idempotency

**Problem:** A scheduled task can be restarted halfway through its work.

**Reference approach:** Track durable progress and make each unit of work safely repeatable. If cluster exclusivity is required, combine a distributed coordination strategy with idempotency.

**Validation:** Stop the process mid-batch and verify restart resumes safely.

## Exercise 81 — API rate limit

**Problem:** Apply a quota per authenticated client.

**Reference approach:** Use a gateway or shared limiter keyed by a trusted client identity, with explicit burst and failure semantics.

**Validation:** Test distributed concurrency and ensure spoofed headers cannot change identity.

## Exercise 82 — Request timeout budget

**Problem:** An endpoint makes sequential and parallel downstream calls.

**Reference approach:** Derive a total deadline and allocate bounded per-call budgets; avoid nested retry loops that exceed the caller's deadline.

**Validation:** Inject slow calls and assert total response time remains within the documented bound.

## Exercise 83 — Secure Actuator

**Problem:** Expose health publicly but restrict environment and configuration endpoints.

**Reference approach:** Configure endpoint exposure and security rules explicitly for the chosen Boot version and deployment model.

**Validation:** Test unauthenticated access to each exposed endpoint and ensure sensitive details are not returned.

## Exercise 84 — Kubernetes readiness behavior

**Problem:** Keep a starting instance out of service until it can safely handle requests.

**Reference approach:** Configure readiness and startup probes according to startup duration and application readiness semantics; avoid making liveness depend on every external dependency.

**Validation:** Simulate slow startup and a transient dependency outage.

## Exercise 85 — Upgrade dependency BOM

**Problem:** Resolve an incompatible transitive dependency after Boot upgrade.

**Reference approach:** Inspect the resolved dependency tree, align the Boot BOM and compatible Cloud train, and override only with a documented reason.

**Validation:** Run startup, security, persistence, and client smoke tests against the resolved graph.
