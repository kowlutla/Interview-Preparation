# Reference Solution — Actuator Health and Metrics

## Health indicator

```java
@Component
public class InventoryHealthIndicator implements HealthIndicator {
    private final InventoryProbe probe;

    public InventoryHealthIndicator(InventoryProbe probe) {
        this.probe = probe;
    }

    @Override
    public Health health() {
        try {
            probe.checkWithTimeout();
            return Health.up().withDetail("dependency", "inventory").build();
        } catch (Exception ex) {
            // Do not return exception messages or secrets in public health details.
            return Health.down().withDetail("dependency", "inventory").build();
        }
    }
}
```

The probe must have a strict timeout and minimal work. Decide whether inventory failure affects readiness; do not automatically make it a liveness failure or create a restart loop during a shared outage. Secure and expose Actuator endpoints deliberately.

## Timer and counter

```java
@Service
public class CheckoutMetrics {
    private final MeterRegistry registry;

    public CheckoutMetrics(MeterRegistry registry) {
        this.registry = registry;
    }

    public <T> T record(String outcome, Supplier<T> operation) {
        return Timer.builder("checkout.operation")
            .tag("outcome", outcome)
            .register(registry)
            .record(operation);
    }

    public void recordFailure(String category) {
        Counter.builder("checkout.failures")
            .tag("category", category)
            .register(registry)
            .increment();
    }
}
```

In production, keep tag values bounded. Do not use order IDs, user IDs, full URLs, or exception messages as metric tags. Ensure the timer records the actual outcome rather than passing a hard-coded value that can become inaccurate.

## Validation plan

- [ ] Health probe times out when the dependency is slow.
- [ ] Dependency outage affects readiness only if that is the intended policy.
- [ ] Public health response does not expose internal exception details.
- [ ] Timer captures duration and bounded outcome categories.
- [ ] Failure counter increments for known categories.
- [ ] Dashboards correlate metrics with logs/traces and deployment version.
