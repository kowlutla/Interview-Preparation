# Advanced Questions — Messaging, Cloud, and Deployment

## Q&A

### Q1. What is an application event's delivery guarantee?
Ordinary Spring application events are in-process and not durable across process crashes. Use a durable broker/outbox when delivery must survive process failure.

### Q2. What does @TransactionalEventListener(AFTER_COMMIT) guarantee?
It ties listener execution to the transaction commit phase when a transaction is active. It does not durably queue work after commit if the process crashes.

### Q3. Which failure window does an outbox close, and which failure window remains?
Writing business state and an outbox record in one local transaction closes the gap where one commits but the other never exists. It does not remove the publish-then-crash-before-marking-sent window; the relay can publish duplicates. Design stable event identity, retries, and consumer idempotency for that remaining window.

### Q4. Does an outbox guarantee exactly-once publication?
No. A relay can publish and crash before recording success, causing duplicate publication. Consumers should be idempotent.

### Q5. What durable state is needed to make a message consumer idempotent?
Persist a stable event or command identity and enforce uniqueness at the same consistency boundary as the business state change. In-memory deduplication is lost on restart and can fail under concurrency. If the effect is an external API call, a local deduplication row alone cannot make that remote side effect atomic; use provider idempotency or reconciliation.

### Q6. What is a dead-letter topic?
A destination for messages that cannot be processed under the configured policy, enabling inspection and controlled replay. It requires monitoring and an operational ownership process.

### Q7. Why should retries distinguish poison messages?
Permanent schema or business failures will not heal with retries and can block progress or waste capacity. Route non-retryable failures appropriately.

### Q8. What is a consumer group?
A set of consumers cooperating to divide partitions of a topic. Parallelism is constrained by partition assignment and ordering semantics.

### Q9. Why does Kafka ordering depend on partitioning?
Ordering is guaranteed within a partition, not globally across all partitions. Choose keys that align with the entity/order requirement.

### Q10. What can happen during a consumer rebalance?
Partitions are reassigned, potentially interrupting work and causing reprocessing depending on acknowledgment/commit behavior. Processing should tolerate duplicates.

### Q11. Why does batch consumption need special care?
Batching changes failure granularity, memory use, acknowledgment behavior, and replay cost. Define how partially processed batches are handled.

### Q12. What is Spring Cloud Config for?
It provides centralized external configuration integration in supported setups. Availability, refresh behavior, security, and bootstrap/import mechanics depend on version and configuration.

### Q13. Does refreshing configuration update every bean?
No. Only supported refresh-aware components and properties may update, and refresh semantics are integration-specific. Restart/rollout may be safer.

### Q14. What is service discovery's failure mode?
Registry staleness or availability issues can lead to missing or dead instances. Clients need bounded timeouts and platform-appropriate recovery.

### Q15. What does client-side load balancing do?
It selects among available service instances from a client-side view. It does not fix slow service behavior or remove the need for health, timeout, and overload policies.

### Q16. What is a gateway's main security risk?
Misconfigured routes, trusted forwarded headers, or direct service exposure can bypass intended edge controls. Enforce resource authorization in the service as well.

### Q17. Why must Spring Cloud versions align with Boot?
Cloud integrations depend on specific framework and Boot APIs. Incompatible trains can fail at compile time or runtime.

### Q18. What is graceful termination in Kubernetes?
The platform signals termination and later forcefully kills the process after a grace period. Application shutdown hooks and probe/endpoint behavior must fit inside that period.

### Q19. Why should a container not run as root by default?
Least privilege reduces the impact of application compromise. Use a non-root runtime identity and restrict filesystem/capabilities where feasible.

### Q20. Why externalize environment configuration?
It allows one artifact to run across environments with controlled settings. Secrets still require access control and rotation, not merely environment variables.

### Q21. What is an expand-and-contract migration?
First deploy a schema compatible with old and new code, migrate application usage, then remove old schema in a later step. It supports rolling deployments where versions coexist.

### Q22. Why can startup probes be useful?
They allow slow-starting applications time to initialize before liveness checks begin causing restarts. Probe settings must reflect actual startup distributions.

### Q23. What is the difference between startup, readiness, and liveness probes?
Startup determines whether initialization has completed; readiness controls traffic eligibility; liveness detects a process that should be restarted. Their exact orchestration behavior depends on platform configuration.

### Q24. Why budget memory beyond the heap?
Native memory, thread stacks, metaspace, direct buffers, and libraries also consume container memory.

### Q25. What is a safe rollout rollback?
A rollback returns application code/configuration to a known-good version while accounting for database schema and irreversible side effects. Database changes must be backward-compatible or have a recovery plan.

### Q26. Why is a container image not sufficient for reproducibility?
Runtime configuration, external services, secrets, platform settings, and mutable tags can still vary. Pin artifacts and document runtime dependencies.

### Q27. What is SBOM useful for?
A software bill of materials records included components and helps vulnerability and license management. It complements dependency scanning and patch processes.

### Q28. Why should secrets be rotated?
Rotation limits exposure duration and supports response to compromise. Applications must handle credential updates safely and avoid logging secrets.

### Q29. What is a production readiness review?
A structured review of reliability, security, observability, capacity, recovery, deployment, and ownership before release.

### Q30. What is the most important deployment invariant?
New and old versions must coexist safely during rollout, including API contracts, schema compatibility, message formats, and configuration.
