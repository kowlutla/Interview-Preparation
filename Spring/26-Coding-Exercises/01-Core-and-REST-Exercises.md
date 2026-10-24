# Coding Exercises — Core Spring and REST

Each exercise states the task, expected behavior, a reference approach, and validation points. Examples assume a modern Spring Boot 3.x baseline with Java 17+ unless the project pins a different supported version; adapt dependencies to the target release.

## Exercise 1 — Constructor injection

**Problem:** Implement `PriceService` using a `TaxPolicy` interface.

**Requirements:** The service must not instantiate its policy; tests must be able to supply a fake policy.

**Reference solution:**
```java
public interface TaxPolicy {
    BigDecimal taxFor(BigDecimal net);
}
@Service
public class PriceService {
    private final TaxPolicy policy;
    public PriceService(TaxPolicy policy) { this.policy = policy; }
    public BigDecimal gross(BigDecimal net) {
        return net.add(policy.taxFor(net));
    }
}
```

**Why it works:** The dependency is explicit and replaceable. Production configuration must register one unambiguous TaxPolicy bean.

**Common mistake:** Constructing a concrete policy inside the service.

**Validation:** Unit-test with a lambda or fake implementation; start the context to verify wiring.

## Exercise 2 — Resolve multiple implementations

**Problem:** Register `CardPayment` and `BankPayment` implementations and inject the intended one into checkout.

**Expected behavior:** Checkout selects by a semantic qualifier, not incidental registration order.

**Reference approach:** Add a qualifier annotation or use `@Qualifier("cardPayment")` on the injection point and matching bean.

**Common mistake:** Expecting Spring to choose the first discovered bean.

**Validation:** Context test verifies the chosen implementation and that an unqualified ambiguous injection fails.

## Exercise 3 — Type-safe configuration

**Problem:** Bind a payment endpoint URL and request timeout.

**Reference solution:**
```java
@ConfigurationProperties(prefix = "payments")
@Validated
public record PaymentProperties(
    @NotNull URI baseUrl,
    @NotNull Duration timeout) {}
```
Enable configuration-properties scanning or register the type explicitly, and include validation support. Configure canonical keys such as `payments.base-url` and `payments.timeout`.

**Common mistake:** Keeping related settings scattered across many `@Value` fields.

**Validation:** Test valid binding and startup failure for malformed or missing required values.

## Exercise 4 — REST endpoint with DTOs

**Problem:** Implement `POST /api/orders` without exposing a persistence entity.

**Reference approach:** Define a request DTO with constraints, a response DTO, a service boundary, and return `ResponseEntity` with an appropriate creation status and location when applicable.

**Common mistake:** Binding the persistence entity directly from untrusted JSON.

**Validation:** Test valid input, invalid fields, missing body, and expected status/response shape with MockMvc.

## Exercise 5 — Global exception mapping

**Problem:** Map an `OrderNotFoundException` to a stable 404 response.

**Reference approach:** Create a narrowly scoped `@RestControllerAdvice` handler and return a documented error schema or supported `ProblemDetail`.

**Common mistake:** Catching every exception and returning 400.

**Validation:** Test status, safe message, correlation identifier, and absence of stack traces.

## Exercise 6 — Custom validation

**Problem:** Validate that a requested delivery date is not before today in the business time zone.

**Reference approach:** Implement a Jakarta Bean Validation constraint and validator, inject a Clock if supported by the chosen validator setup, and keep the rule testable.

**Common mistake:** Duplicating the rule in multiple controllers.

**Validation:** Test boundary dates, time-zone transitions relevant to the business, and invalid values.

## Exercise 7 — Pagination and stable sorting

**Problem:** Expose `GET /api/orders?page=&size=&sort=`.

**Reference approach:** Validate page size bounds, whitelist sort fields, use Spring Data Pageable, and include a deterministic unique tie-breaker.

**Common mistake:** Accepting arbitrary sort expressions or unbounded page sizes.

**Validation:** Test empty results, maximum size, invalid sort field, and stable ordering for equal timestamps.

## Exercise 8 — Conditional bean

**Problem:** Provide a default notification sender only when the application has not supplied one.

**Reference approach:** Put a `@ConditionalOnMissingBean(NotificationSender.class)` factory method in Boot auto-configuration, and register it using the metadata format for the selected Boot version.

**Common mistake:** Treating the condition as a replacement for a proper library registration test.

**Validation:** Test both default creation and application override.

## Exercise 9 — Profile-specific implementation

**Problem:** Use a local stub only in a development profile.

**Reference approach:** Isolate the stub bean with `@Profile("dev")`, keep production configuration explicit, and fail startup if the production implementation is missing.

**Common mistake:** Making a permissive stub the unconditional fallback.

**Validation:** Start contexts with dev and prod profiles and assert exactly the intended bean exists.

## Exercise 10 — Bean lifecycle resource

**Problem:** Create a managed component that owns a resource requiring cleanup.

**Reference approach:** Use a supported lifecycle callback or an explicit resource abstraction with deterministic close semantics. Keep initialization bounded and make cleanup idempotent.

**Common mistake:** Assuming cleanup callbacks run after SIGKILL or machine failure.

**Validation:** Close the context and assert cleanup occurs; document abrupt-termination limitations.

## Exercise 11 — Transaction rollback

**Problem:** Debit one account and credit another in one database transaction.

**Reference approach:** Place the operation behind a proxied service transaction boundary, validate both accounts, and allow failures requiring rollback to escape the boundary or configure explicit rollback rules.

**Common mistake:** Catching and swallowing the exception inside the transaction.

**Validation:** Force failure after the first update and verify both balances remain unchanged.

## Exercise 12 — Avoid self-invocation

**Problem:** A method annotated with `@Transactional` is called by another method on the same object and does not start the expected transaction.

**Reference solution:** Move the transactional operation to a separate injected service with a clear boundary, or restructure the call so it passes through the proxy.

**Common mistake:** Assuming annotations intercept every Java method call.

**Validation:** Verify transaction activity and rollback in an integration test.

## Exercise 13 — MockMvc validation test

**Problem:** Test that invalid order input returns a 400 response.

**Reference approach:** Use `@WebMvcTest`, MockMvc, a mocked application service, and an assertion on the stable error contract.

**Common mistake:** Mocking the controller itself, which bypasses request processing.

**Validation:** Include valid and invalid payload tests, plus malformed JSON.

## Exercise 14 — Secured endpoint

**Problem:** Require an authority for `DELETE /api/orders/{id}`.

**Reference approach:** Define a SecurityFilterChain request rule or method-level authorization, use least-privilege authorities, and test anonymous, authenticated-but-forbidden, and authorized requests.

**Common mistake:** Testing only the success path.

**Validation:** Verify 401/403 behavior according to the configured entry point and access-denied handler.

## Exercise 15 — JWT resource server

**Problem:** Protect an API with signed JWT access tokens from a trusted issuer.

**Reference approach:** Configure Spring Security OAuth2 Resource Server with issuer/JWK settings; define audience validation if the deployment's trust contract requires it.

**Common mistake:** Decoding a token and trusting claims without signature validation.

**Validation:** Test valid token, expired token, wrong issuer, malformed token, and insufficient authority.

## Exercise 16 — RestClient timeout

**Problem:** Call an inventory service with bounded latency.

**Reference approach:** Configure connection and response timeouts on the chosen request factory, define a total deadline, map downstream failures, and instrument duration.

**Common mistake:** Retrying every exception without a budget.

**Validation:** Use a test server that delays or refuses connections and assert bounded failure behavior.

## Exercise 17 — Cache invalidation

**Problem:** Cache product reads and invalidate after a product update.

**Reference approach:** Define stable cache keys and evict/update the correct key only after the write succeeds. Ensure the method invocation crosses the cache proxy.

**Common mistake:** Invalidating a different key format than the read path uses.

**Validation:** Read twice to verify a hit, update, then verify fresh data is returned.

## Exercise 18 — Bounded async executor

**Problem:** Execute background notifications without unbounded queue growth.

**Reference approach:** Configure an explicit ThreadPoolTaskExecutor with worker and queue limits, a rejection policy, thread naming, metrics, and graceful shutdown.

**Common mistake:** Using an unbounded queue and assuming @Async increases capacity infinitely.

**Validation:** Saturate the executor and verify rejection/backpressure behavior is observable.

## Exercise 19 — Scheduled job lock

**Problem:** Ensure a cleanup job does not run concurrently across replicas.

**Reference approach:** Make work idempotent and use a distributed lock or scheduler designed for clustered execution. A local scheduler alone does not coordinate replicas.

**Common mistake:** Using synchronized as a cluster-wide lock.

**Validation:** Run two instances against shared coordination storage and simulate lock expiry/failure.

## Exercise 20 — Transactional outbox

**Problem:** Persist an order and an event reliably.

**Reference approach:** Write order state and an outbox row in one local transaction; publish pending rows asynchronously with retries and idempotent downstream handling.

**Common mistake:** Saving to the database and publishing directly to Kafka as two uncoordinated writes.

**Validation:** Simulate process failure before and after publish, then verify eventual delivery and duplicate safety.

## Exercise 21 — Actuator health indicator

**Problem:** Add a bounded health check for a critical dependency.

**Reference approach:** Implement a HealthIndicator or reactive equivalent appropriate to the application, use a short timeout, and decide whether the dependency affects readiness or liveness.

**Common mistake:** Performing an expensive unbounded query on every probe.

**Validation:** Test healthy, unavailable, and timed-out dependency responses.

## Exercise 22 — Testcontainers integration test

**Problem:** Verify repository behavior against the production database engine.

**Reference approach:** Start a disposable container with a pinned compatible image, configure dynamic connection properties, apply migrations, and clean test state between cases.

**Common mistake:** Assuming H2 reproduces production SQL, locking, and collation behavior.

**Validation:** Run in CI and confirm container startup and cleanup behavior.

## Exercise 23 — Reactive non-blocking flow

**Problem:** Compose two downstream calls in WebFlux.

**Reference approach:** Use WebClient and Reactor composition operators, propagate errors intentionally, and avoid calling blocking APIs on event-loop threads.

**Common mistake:** Calling `block()` inside a reactive controller.

**Validation:** Test success, timeout, cancellation, and error mapping with StepVerifier or WebTestClient.

## Exercise 24 — Custom metrics

**Problem:** Measure order-processing duration and failures.

**Reference approach:** Use Micrometer Timer and Counter instruments with low-cardinality dimensions such as operation and outcome.

**Common mistake:** Adding order IDs or user IDs as metric tags.

**Validation:** Assert instrumentation through a test registry and check cardinality under representative traffic.

## Exercise 25 — Diagnose startup failure

**Problem:** An application fails with NoSuchBeanDefinitionException after moving a class to another package.

**Reference approach:** Check the component-scan root, bean conditions, active profiles, runtime classpath, and the failing test context. Move the application class to a suitable root package or configure scanning deliberately.

**Common mistake:** Adding a second component scan without understanding package boundaries.

**Validation:** Add a context-start test and verify the intended bean is present exactly once.
