# Coding Exercises — Testing and Integration

## Exercise 51 — Service unit test

**Problem:** Test a business service without starting Spring.

**Reference approach:** Construct the service directly with a fake or Mockito mock for its collaborator. Assert outcomes and interactions only where they are part of the contract.

**Common mistake:** Loading the entire application context for a pure calculation.

**Validation:** Include success, invalid input, and collaborator failure.

## Exercise 52 — MVC slice test

**Problem:** Test controller binding, validation, and response serialization.

**Reference approach:** Use `@WebMvcTest`, MockMvc, and mocked application-layer collaborators.

**Related detailed solution:** [Validated REST API and MockMvc](Reference-Solutions/01-REST-API-Reference-Solution.md)

**Validation:** Assert status, JSON fields, validation response, and unsupported media behavior.

## Exercise 53 — JPA slice test

**Problem:** Verify a repository query and pagination behavior.

**Reference approach:** Use `@DataJpaTest` for focused persistence configuration, but run database-specific tests against the production engine where semantics matter.

**Related detailed solution:** [Testcontainers and HTTP client patterns](Reference-Solutions/05-Testcontainers-and-Client-Reference.md)

**Validation:** Cover empty data, sorting ties, query predicates, and transaction cleanup.

## Exercise 54 — JSON serialization test

**Problem:** Ensure a public DTO serializes to the expected API contract.

**Reference approach:** Use the supported JSON test slice and assert stable fields, date formats, null handling, and unknown-field policy as appropriate.

**Validation:** Include backward-compatibility expectations for fields consumed by clients.

## Exercise 55 — Rest client test

**Problem:** Test outbound request path, headers, payload, and error mapping.

**Reference approach:** Use the supported client test slice or a controlled mock HTTP server. Avoid real external network dependencies in unit tests.

**Related detailed solution:** [Testcontainers and RestClient patterns](Reference-Solutions/05-Testcontainers-and-Client-Reference.md)

**Validation:** Assert timeout and non-2xx handling as well as success.

## Exercise 56 — Test context cache

**Problem:** Reduce a slow test suite without losing confidence.

**Reference approach:** Reuse consistent context configurations, avoid unnecessary unique test properties, and use slices for narrow behavior. Do not share mutable global state between tests.

**Validation:** Compare test runtime and confirm tests remain isolated under parallel execution if enabled.

## Exercise 57 — Test scheduled work

**Problem:** Verify a scheduled task's business logic deterministically.

**Reference approach:** Separate the task method from the scheduling trigger and call the business operation directly in unit tests. Use controlled scheduler integration tests only for cadence semantics.

**Related detailed solution:** [Scheduled work and resilience](Reference-Solutions/09-Scheduled-Work-and-Resilience.md)

**Validation:** Verify duplicate invocation is safe and failures are observable.

## Exercise 58 — Test Kafka listener failure

**Problem:** Verify retry and dead-letter behavior.

**Reference approach:** Use the Spring Kafka test support or a broker container when broker semantics matter. Make the failure deterministic and inspect the resulting retry/dead-letter path.

**Related detailed solution:** [Transactional outbox and idempotent consumer](Reference-Solutions/07-Outbox-and-Idempotency.md)

**Validation:** Assert successful retry, permanent failure routing, and duplicate-safe business effect.

## Exercise 59 — Test transaction rollback

**Problem:** Verify a failed service operation leaves no partial database state.

**Reference approach:** Invoke through the Spring-managed proxy in an integration test and assert database state after the exception. Ensure the test itself does not mask the behavior through its own surrounding transaction.

**Related detailed solution:** [Transactional transfer and rollback testing](Reference-Solutions/02-Transaction-Reference-Solution.md)

**Validation:** Include checked-exception behavior if the application uses explicit rollback rules.

## Exercise 60 — Test after-commit events

**Problem:** Verify a transactional listener only reacts after commit.

**Reference approach:** Publish the event within a transaction and test both commit and rollback cases. Use transaction-test utilities appropriate to the test context.

**Related detailed solution:** [Transactional outbox and idempotent consumer](Reference-Solutions/07-Outbox-and-Idempotency.md)

**Validation:** Confirm external side effects are not attempted after rollback.

## Exercise 61 — Test JWT claim validation

**Problem:** Verify resource-server issuer and authority mapping.

**Reference approach:** Test the configured decoder/validators or use Spring Security test support for endpoint authorization. Include invalid issuer, expired token, and missing authority cases.

**Related detailed solution:** [JWT resource-server configuration and authorization](Reference-Solutions/03-Security-Reference-Solution.md)

**Validation:** Ensure a token with valid signature but wrong trust claims is rejected.

## Exercise 62 — Test CORS policy

**Problem:** Allow only approved browser origins and methods.

**Reference approach:** Configure CORS in the appropriate security/MVC layer and test preflight requests, allowed origin, denied origin, and credentials behavior.

**Related detailed solution:** [JWT resource-server configuration and authorization](Reference-Solutions/03-Security-Reference-Solution.md)

**Validation:** Confirm CORS is not treated as authorization for non-browser clients.

## Exercise 63 — Test application startup conditions

**Problem:** Verify a required bean exists only when a feature property is enabled.

**Reference approach:** Use isolated application-context tests or Boot condition testing support to cover both property states.

**Validation:** Assert bean presence/absence and startup failure when a required dependency is missing.

## Exercise 64 — Test shutdown hooks

**Problem:** Verify an executor stops accepting work and waits within a configured bound.

**Reference approach:** Create the context, submit controlled tasks, close the context, and assert the documented shutdown policy.

**Related detailed solution:** [Scheduled work and resilience](Reference-Solutions/09-Scheduled-Work-and-Resilience.md)

**Validation:** Include a task that exceeds the grace period and ensure shutdown does not hang indefinitely.

## Exercise 65 — Test observability

**Problem:** Verify a timer and failure counter are recorded for a service operation.

**Reference approach:** Use a simple MeterRegistry in a focused test and assert instrument updates without depending on a production monitoring server.

**Related detailed solution:** [Actuator health and metrics](Reference-Solutions/08-Actuator-and-Metrics.md)

**Validation:** Ensure metric tags are bounded and do not contain request-specific identifiers.

## Exercise 66 — Test property validation

**Problem:** Fail fast when a timeout is zero or negative.

**Reference approach:** Bind the configuration-properties type in a context test with validation enabled.

**Related detailed solution:** [Configuration binding and custom validation](Reference-Solutions/06-Configuration-and-Validation.md)

**Validation:** Test missing, malformed, and boundary values with clear startup diagnostics.

## Exercise 67 — Test API idempotency

**Problem:** Repeated POST requests with the same idempotency key must not create duplicate orders.

**Reference approach:** Persist the key and result under a uniqueness constraint and define concurrent-request behavior.

**Related detailed solution:** [Transactional outbox and idempotent consumer](Reference-Solutions/07-Outbox-and-Idempotency.md)

**Validation:** Send identical requests sequentially and concurrently; assert one order and deterministic responses.

## Exercise 68 — Test connection-pool saturation

**Problem:** Demonstrate how slow database work affects request latency.

**Reference approach:** Use a controlled integration test environment with a deliberately small pool and slow query, then inspect pool metrics and timeout behavior.

**Validation:** Ensure the test is isolated and bounded so it cannot hang CI.

## Exercise 69 — Test graceful degradation

**Problem:** Return a documented degraded response when a noncritical recommendation service is down.

**Reference approach:** Define the fallback contract in the application layer and use bounded client timeouts. Do not fabricate fallback values for critical authorization or financial operations.

**Related detailed solution:** [Scheduled work and resilience](Reference-Solutions/09-Scheduled-Work-and-Resilience.md)

**Validation:** Test downstream timeout, fallback response, metric/log emission, and recovery.

## Exercise 70 — Test trace propagation

**Problem:** Verify trace context reaches an outbound client call.

**Reference approach:** Use supported tracing instrumentation and a mock server that captures headers.

**Related detailed solution:** [Actuator health and metrics](Reference-Solutions/08-Actuator-and-Metrics.md)

**Validation:** Assert context propagation without coupling tests to unrelated generated IDs or exact sampling decisions.
