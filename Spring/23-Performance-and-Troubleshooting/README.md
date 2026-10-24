# Performance and Troubleshooting

## Q&A

### Q1. How do you investigate a slow Spring REST endpoint?

Break latency into server queueing, application processing, database calls, serialization, and downstream HTTP calls. Use request metrics, traces, SQL timings, thread dumps, and load tests. Avoid attributing the issue to Spring before identifying where time is spent.

### Q2. What does a thread dump reveal?

It shows thread states and stack traces at a point in time, helping identify blocked threads, lock contention, deadlocks, saturated workers, and blocking calls on reactive event loops. Capture multiple dumps when useful to distinguish persistent blocking from transient activity.

### Q3. How do you diagnose connection pool exhaustion?

Correlate active/pending connection metrics with transaction duration, slow queries, request concurrency, and thread stacks. Check for connections held while doing remote calls or waiting on locks. Increasing the pool without checking database capacity may make the incident worse.

### Q4. How should memory growth be investigated?

Compare heap usage after garbage collection, allocation rate, object histograms, heap dumps, and container/native memory metrics. Look for retained collections, unbounded caches or queues, listener registrations, and thread-local/context leaks. A high heap before GC alone does not prove a leak.

### Q5. What causes excessive Spring application startup time?

Expensive bean initialization, broad classpath scanning, remote calls during startup, migrations, large dependency graphs, and eager singleton creation can contribute. Measure before optimizing, and verify that lazy initialization does not simply move failures and latency into production requests.

### Q6. How do you distinguish JVM, database, framework, and downstream bottlenecks?

Combine evidence: CPU and GC metrics, thread states, pool wait times, query plans, trace spans, and application timings. A framework stack frame in a trace does not prove the framework is the root cause; it may be waiting on the database or a remote service.

### Q7. What is a safe performance optimization process?

Establish a representative baseline, define a measurable target, change one major factor at a time, and compare throughput, latency percentiles, errors, resource use, and correctness. Keep a rollback path and avoid tuning based on a single local microbenchmark when production behavior is dominated by I/O or contention.

## Official references

- [Spring Boot Actuator metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
- [Java Flight Recorder command-line tool (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jfr.html)
