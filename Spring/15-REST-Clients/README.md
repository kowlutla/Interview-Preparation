# REST Clients and Inter-Service Communication

## Q&A

### Q1. How do RestTemplate, RestClient, and WebClient differ?

RestTemplate is the established synchronous template-style client; it is deprecated in Spring Framework 7.0 in favor of RestClient. RestClient is a fluent synchronous client introduced in Framework 6.1. WebClient is designed for non-blocking reactive I/O and streaming. Verify the framework baseline before using RestClient; do not assume it exists in older Framework versions.

For new synchronous integrations on Framework 6.1+, evaluate RestClient. For non-blocking reactive pipelines, consider WebClient. Existing RestTemplate applications should follow the official migration guide and can migrate incrementally.

### Q2. How should outbound timeouts be configured?

Configure connection establishment and response/read timeouts using the chosen client and request factory/connector. A timeout budget should align with the caller's deadline and leave time for response handling. An absent timeout can tie up threads and connections during a downstream incident.

### Q3. When are retries appropriate?

Retries can help with transient failures when the operation is safe to repeat and the total deadline allows it. Use bounded attempts, backoff, jitter, and clear retryable-exception rules. Retrying non-idempotent writes without deduplication can create duplicate side effects and amplify an outage.

### Q4. Why is connection pooling important?

Pooling reuses connections and reduces repeated connection setup, while pool limits provide a resource boundary. Size the pool together with request concurrency, connection-acquisition timeout, downstream capacity, and expected request duration; remember that the total across all application replicas matters. A pool that is too small creates waiting, while an oversized pool can overload the dependency. Monitor active, idle, pending, and acquisition-latency metrics and load-test realistic traffic.

### Q5. How should authentication be propagated between services?

Choose an explicit trust model: separate service credentials, token exchange, or narrowly scoped delegated user tokens depending on the downstream authorization requirement. Do not forward an inbound bearer token to every destination simply because the request originated in your service. Allow-list trusted destinations, review redirect behavior, avoid putting credentials in URLs, redact authorization headers from logs, and define token audience/scope so a leaked token cannot be reused against unrelated services.

### Q6. How should client errors be observed?

Capture latency, outcome, timeout/retry counts, and dependency identity with low-cardinality metric labels. Add distributed tracing when supported and avoid logging sensitive request bodies. Separate caller-side timeouts from server-side error responses during diagnosis.

### Q7. What are HTTP Service Clients?

Modern Spring Framework versions support declarative HTTP service interfaces backed by a configured client adapter. They can reduce boilerplate for stable service contracts, but still need explicit timeout, authentication, error mapping, and observability configuration. Verify availability and configuration details for the target Framework release.

## Official references

- [Spring Framework REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
- [Spring Framework 6.1 RestClient announcement](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient/)
- [Spring Framework HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
