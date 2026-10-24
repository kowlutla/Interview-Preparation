# Asynchronous Processing and Scheduling

## Q&A

### Q1. What does @Async do?

When async execution is enabled and the invocation passes through the relevant proxy, Spring submits the method execution to a configured executor. It does not guarantee more capacity or faster completion; executor sizing, queue bounds, rejection policy, and shutdown behavior determine operational safety.

### Q2. Why should thread pools have bounded queues?

An unbounded queue can accumulate work faster than workers process it, increasing memory use and latency until the service becomes unhealthy. Choose worker count and queue capacity based on workload, downstream limits, and acceptable queueing delay. Define what happens when the system is saturated.

### Q3. How are exceptions from asynchronous methods observed?

Exceptions from fire-and-forget void methods require an appropriate async exception handler because the caller cannot receive them. Methods returning Future or CompletableFuture can surface failures when the result is observed. Ensure failures are logged and measured without duplicating logs at every layer.

### Q4. What is the difference between fixed rate and fixed delay scheduling?

Fixed rate schedules executions according to a target cadence; fixed delay measures the delay after an execution completes before the next run. Actual behavior depends on scheduler configuration and task duration. Long tasks, process pauses, and multiple application instances require deliberate concurrency planning.

### Q5. Why can a scheduled job run more than once in a cluster?

Each application replica may register and execute the same schedule. If only one execution is allowed globally, use a distributed coordination/locking mechanism or a scheduler designed for clustered execution, and make the task idempotent. A local synchronized block only coordinates threads in one process.

### Q6. How should transactions interact with async work?

A transaction bound to the calling thread generally does not automatically propagate into a separately executed async task. Define a transaction boundary inside the asynchronous operation if required, and pass immutable data rather than sharing mutable persistence state across threads.

### Q7. How should cron time zones be handled?

Specify the intended zone explicitly when supported by the scheduling annotation/configuration. Test daylight-saving transitions and cluster consistency, and avoid assuming the container's default time zone matches the business zone.

## Official references

- [Spring task execution and scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
