# Deployment and Production Readiness

## Q&A

### Q1. What should a production-ready Spring Boot service include?

At minimum, it needs secure external configuration, bounded resource pools, meaningful readiness/liveness semantics, timeouts for remote dependencies, observable errors and latency, graceful shutdown, tested data migrations, and a rollback strategy. Exact requirements depend on the service's risk and workload.

### Q2. How should a deployment drain requests and background work before termination?

Coordinate readiness removal, server graceful shutdown, consumer pause/stop, in-flight request deadlines, and the platform termination grace period. Consumers should acknowledge only after their work is safely committed, and durable jobs need checkpoints or retries if the process is terminated. Test the full sequence under load; a server's graceful-shutdown setting alone does not protect every background task.

### Q3. How should database pool size be selected?

Estimate concurrent database demand and compare it with database connection limits and query duration. More connections can increase contention and overwhelm the database. Monitor active, idle, pending, acquisition latency, and query timings, then load-test realistic concurrency.

### Q4. Why should container memory limits include more than heap?

The process uses native memory for thread stacks, metaspace, direct buffers, libraries, and other runtime structures in addition to Java heap. Set JVM/container sizing based on measurements and leave headroom for non-heap usage; a heap below the container limit does not guarantee the process will avoid an OOM kill.

### Q5. How should a service handle a dependency outage?

Bound waits with timeouts, limit retries, use circuit breaking when appropriate, shed or queue work within explicit bounds, and return a meaningful failure or degraded response. Decide which functions can operate without the dependency and how recovery is detected.

### Q6. What should be checked before a rolling deployment?

Verify readiness semantics, backward-compatible API and schema changes, graceful termination, consumer rebalance behavior, startup time, and capacity while old and new replicas coexist. Use expand-and-contract database migrations when multiple versions may run simultaneously.

## Official references

- [Spring Boot production-ready features](https://docs.spring.io/spring-boot/reference/actuator/)
- [Spring Boot container images](https://docs.spring.io/spring-boot/reference/packaging/container-images/)
