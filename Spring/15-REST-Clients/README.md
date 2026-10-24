# REST Clients and Inter-Service Communication

## Q&A

### Q1. How do RestTemplate, RestClient, and WebClient differ?

`RestTemplate` is the established synchronous client API; `RestClient` is a newer fluent synchronous API in modern Spring Framework versions; `WebClient` supports non-blocking reactive I/O and can also be used in other interaction styles. Verify framework baseline and support status before selecting one. Do not choose WebClient solely because it is newer.

### Q2. How should outbound timeouts be configured?

Configure connection establishment and response/read timeouts using the chosen client and request factory/connector. A timeout budget should align with the caller's deadline and leave time for response handling. An absent timeout can tie up threads and connections during a downstream incident.

### Q3. When are retries appropriate?

Retries can help with transient failures when the operation is safe to repeat and the total deadline allows it. Use bounded attempts, backoff, jitter, and clear retryable-exception rules. Retrying non-idempotent writes without deduplication can create duplicate side effects and amplify an outage.

### Q4. Why is connection pooling important?

Pooling reuses established connections and bounds resource usage. Pool capacity must be considered with thread concurrency, downstream limits, and request duration. A large pool does not fix a slow dependency and may overwhelm it.

### Q5. How should authentication be propagated between services?

Use an explicit trust model: service credentials, token exchange, or carefully scoped delegated tokens. Do not blindly forward inbound user tokens to arbitrary destinations. Prevent credential leakage through redirects, logs, and untrusted URLs.

### Q6. How should client errors be observed?

Capture latency, outcome, timeout/retry counts, and dependency identity with low-cardinality metric labels. Add distributed tracing when supported and avoid logging sensitive request bodies. Separate caller-side timeouts from server-side error responses during diagnosis.

## Official references

- [Spring Framework REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
- [WebClient](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)
