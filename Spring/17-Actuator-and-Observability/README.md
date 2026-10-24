# Actuator and Observability

## Q&A

### Q1. What does Spring Boot Actuator provide?

Actuator exposes operational endpoints and integrates health, metrics, and application information with the running application. Common uses include readiness/liveness probes, metrics scraping, and inspecting selected runtime state. Endpoint availability is different from web exposure: configure which endpoints are enabled and exposed, secure sensitive endpoints, and avoid publishing environment/configuration details to untrusted callers. Adding Actuator does not mean every endpoint should be public.

### Q2. What is the difference between liveness and readiness?

Liveness asks whether the process should be restarted because it is unable to make progress. Readiness asks whether the instance should receive traffic. A transient dependency failure should not automatically trigger restart loops unless the operational policy explicitly requires that behavior.

### Q3. What is Micrometer?

Micrometer provides an instrumentation facade for metrics that can be exported to supported monitoring systems. Prefer counters, timers, gauges, and histograms that answer operational questions. Avoid high-cardinality labels such as user IDs, raw URLs, or request IDs.

### Q4. How should health indicators be designed?

A health check should reflect a meaningful operational decision and be fast, bounded, and safe. Decide whether each dependency belongs in liveness or readiness, set timeouts, and avoid making a health endpoint itself a source of load during an incident. Health status is not a replacement for metrics and alerts.

### Q5. What is distributed tracing useful for?

Tracing connects spans across service boundaries to show where time is spent and where failures occur. Propagate context using supported instrumentation, sample according to operational needs, and avoid placing secrets or personal data in span attributes. Trace data complements, rather than replaces, logs and metrics.

### Q6. How should production endpoints be secured?

Expose only the endpoints required by the deployment platform and operators, restrict sensitive environment/configuration details, and apply authentication plus network-level controls. Prefer a separate management port/network where appropriate, while remembering that a separate port is not authentication. After upgrades, inspect the effective endpoint exposure and authorization rules in the deployed environment; do not assume prior defaults or security-chain matching still apply.

### Q7. How should custom health indicators avoid cascading failures?

Use short, bounded timeouts and cheap checks; a health indicator should not execute an expensive business workflow or fan out across every downstream dependency. Decide which dependencies are truly required for readiness and report optional degradation separately where possible. During a shared outage, probes must not create a retry storm or turn a partial dependency failure into every instance becoming unready at once.

### Q8. What is the difference between an application metric and a business metric?

Application metrics measure technical behavior such as latency, errors, pool usage, and saturation. Business metrics measure outcomes such as completed orders or failed checkouts. Both are needed because a service can be technically healthy while business success degrades.

### Q9. How do you design alerts from SLOs?

Start with user-visible service-level indicators such as successful requests and latency within a threshold, then choose alert windows that provide actionable response time without excessive noise. Consider multi-window burn-rate alerts for error-budget consumption, and separate symptoms from diagnostic signals. Pair each alert with a dashboard, runbook, and owner; CPU or memory thresholds alone do not always reveal whether users are affected.

### Q10. Why must observability endpoints be reviewed after an upgrade?

Endpoint availability, default exposure, configuration properties, and security integration can change across release lines. After an upgrade, inspect the actual enabled/exposed endpoint set, test anonymous and authorized access, and verify management-port/network policy. Pay particular attention to environment, configuration, heap/thread dumps, and shutdown endpoints because their data or actions can be sensitive. Treat the previous deployment policy as something to re-verify, not as proof of continued safety.

## Official references

- [Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/)
- [Micrometer documentation](https://docs.micrometer.io/)
