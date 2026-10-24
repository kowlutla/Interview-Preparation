# Actuator and Observability

## Q&A

### Q1. What does Spring Boot Actuator provide?

Actuator exposes operational endpoints and integrates health, metrics, and application information with the running application. Endpoint exposure and access must be deliberately configured; adding Actuator does not mean every endpoint should be publicly accessible.

### Q2. What is the difference between liveness and readiness?

Liveness asks whether the process should be restarted because it is unable to make progress. Readiness asks whether the instance should receive traffic. A transient dependency failure should not automatically trigger restart loops unless the operational policy explicitly requires that behavior.

### Q3. What is Micrometer?

Micrometer provides an instrumentation facade for metrics that can be exported to supported monitoring systems. Prefer counters, timers, gauges, and histograms that answer operational questions. Avoid high-cardinality labels such as user IDs, raw URLs, or request IDs.

### Q4. How should health indicators be designed?

A health check should reflect a meaningful operational decision and be fast, bounded, and safe. Decide whether each dependency belongs in liveness or readiness, set timeouts, and avoid making a health endpoint itself a source of load during an incident. Health status is not a replacement for metrics and alerts.

### Q5. What is distributed tracing useful for?

Tracing connects spans across service boundaries to show where time is spent and where failures occur. Propagate context using supported instrumentation, sample according to operational needs, and avoid placing secrets or personal data in span attributes. Trace data complements, rather than replaces, logs and metrics.

### Q6. How should production endpoints be secured?

Expose only the endpoints needed by the platform, restrict sensitive environment/configuration details, and apply authentication/network controls. Review actual exposure configuration after upgrades because endpoint defaults and integration details can change across versions.

## Official references

- [Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/)
- [Micrometer documentation](https://docs.micrometer.io/)
