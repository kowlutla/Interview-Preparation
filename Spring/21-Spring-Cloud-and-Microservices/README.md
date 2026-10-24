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

Use supported instrumentation and trace-context propagation across HTTP and messaging boundaries. Avoid manually inventing incompatible headers, and keep identifiers out of high-cardinality metric labels. Validate traces end to end, including asynchronous processing.

### Q7. What does a load balancer not solve?

It does not make a slow or failing service healthy, guarantee session consistency, or remove the need for timeouts and retries. Health checks, balancing algorithm, connection reuse, request distribution, and overload behavior all affect outcomes.

## Official references

- [Spring Cloud project and compatibility](https://spring.io/projects/spring-cloud)
- [Spring Cloud reference documentation](https://docs.spring.io/spring-cloud/docs/current/reference/html/)
