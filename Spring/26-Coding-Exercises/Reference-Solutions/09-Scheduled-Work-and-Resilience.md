# Reference Solution — Scheduled Work and Resilience

## Separate scheduling from business logic

```java
@Component
public class ReconciliationSchedule {
    private final ReconciliationService service;

    public ReconciliationSchedule(ReconciliationService service) {
        this.service = service;
    }

    @Scheduled(cron = "${reconciliation.cron}", zone = "${reconciliation.zone:UTC}")
    public void run() {
        service.reconcilePendingItems();
    }
}
```

Configure and validate the cron expression and zone for the selected application. A schedule runs in each application instance unless cluster coordination is added. Make the service idempotent and use a distributed lock/scheduler when global exclusivity is required.

## Resilience policy

For outbound operations, define a total deadline, connection/response timeout, retryable failure set, bounded attempt count, backoff/jitter, and concurrency limit. Use a circuit breaker only when its failure semantics are understood. A fallback must preserve domain correctness; critical writes should not report success when the downstream outcome is unknown.

## Failure test matrix

- Dependency succeeds immediately.
- Dependency returns a permanent client error.
- Dependency times out before response.
- Caller deadline expires during retry backoff.
- Concurrent scheduled invocations overlap.
- Two application replicas attempt the same scheduled task.
- Process stops while work is in progress.
- Dependency recovers after the circuit opens.

## Review questions

- Is the operation safe to repeat?
- Can a timeout occur after the remote side effect committed?
- Does the retry budget fit inside the caller's deadline?
- What prevents an unbounded queue?
- How is duplicate work detected?
- Which metrics show saturation and recovery?
