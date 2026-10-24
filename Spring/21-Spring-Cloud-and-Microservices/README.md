# Spring Cloud and Microservices Integration

## Q&A

### Q1. What is Spring Cloud responsible for?

Spring Cloud provides integrations and abstractions for common distributed-system concerns such as configuration, service discovery, client-side load balancing, gateways, and resilience. Spring Framework provides core application infrastructure and Boot simplifies configuration and packaging; Cloud components have their own compatibility and maintenance considerations.

### Q2. Why must Spring Cloud and Boot versions be compatible?

Cloud release trains are tested against specific Boot generations and managed dependency combinations. Arbitrarily mixing versions can cause linkage errors, missing classes, or subtle runtime failures. Use the official compatibility guidance and its dependency-management BOM.

### Q3. What is the role of an API gateway?

A gateway provides a controlled edge for routing and cross-cutting policies such as authentication integration, rate limiting, and request transformation. It should not become the only place business authorization exists, and its failure characteristics and bypass paths must be considered.

### Q4. How should timeouts, retries, and circuit breakers work together?

Timeouts bound individual waits, retries attempt recovery from selected transient failures, and circuit breakers limit repeated calls to an unhealthy dependency. Poorly aligned policies can multiply latency and load. Budget the full call chain, avoid retry storms, and define fallback behavior that remains correct.

### Q5. What is service discovery?

Service discovery resolves logical service identities to available instances through a registry or platform mechanism. It introduces freshness, health, and propagation considerations. In orchestrated environments, platform-native DNS and service abstractions may be sufficient; do not add a registry without a need.

### Q6. How should distributed tracing be propagated?

Use supported Micrometer Tracing/OpenTelemetry instrumentation and propagate standard trace context across HTTP and messaging boundaries. Avoid manually inventing incompatible headers or logging full sensitive baggage. Keep trace/request identifiers out of high-cardinality metric labels; put them in traces or structured logs instead. Validate an end-to-end trace through asynchronous processing and verify context propagation across executor, broker, and consumer boundaries.

### Q7. What does a load balancer not solve?

It does not make a slow or failing service healthy, guarantee session consistency, or remove the need for timeouts and retries. Health checks, balancing algorithm, connection reuse, request distribution, and overload behavior all affect outcomes.

### Q8. What is the difference between a retry and a circuit breaker?

A retry repeats a selected operation after a failure; a circuit breaker suppresses calls when recent outcomes indicate a dependency is unhealthy. Retries can amplify load, while a breaker can reject calls that might have succeeded. Configure both around an explicit deadline and operation semantics.

### Q9. Why can timeout values not be selected independently per service?

A request may traverse a gateway and several nested services, so each hop's processing, connection wait, and retry budget must fit within the caller's remaining deadline. Independently generous timeouts can make end-to-end latency unpredictable and keep upstream threads occupied after the caller has given up. Propagate deadlines where supported, reserve time for response handling, and avoid retries that can outlive the original request's useful lifetime.

### Q10. What is a service mesh's relationship to Spring Cloud?

A service mesh can provide infrastructure-level traffic, identity, and telemetry policies, while Spring Cloud provides application-library integrations. They may overlap; avoid implementing the same retry/security policy at multiple layers without understanding precedence and compounded behavior.

### Q11. How should service discovery and readiness interact?

Discovery should ideally expose instances that are eligible to receive traffic, but propagation and caching mean stale instances can still be observed. Clients need bounded timeouts and failure handling even when health registration is configured correctly.

### Q12. How do you avoid duplicate resilience policies across gateway and service clients?

Assign clear ownership for connection/response timeouts, retries, and rate limits at each hop. A gateway may protect ingress while a service client handles a narrowly defined transient failure, but their combined budgets must be documented. Nested retries multiply attempts and can amplify an outage; use bounded retries with backoff/jitter and test the full call path under latency, throttling, and dependency failure.

## Official references

- [Spring Cloud project and compatibility](https://spring.io/projects/spring-cloud)
- [Spring Cloud release-train reference documentation](https://docs.spring.io/spring-cloud-release/reference/)
